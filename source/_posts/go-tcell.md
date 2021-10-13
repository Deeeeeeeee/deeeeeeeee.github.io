---
title: 【go】tcell终端编辑库
categories:
  - go
tags:
  - go
  - tcell
date: 2021-10-13 17:00:08
---

## 目录

## 

```shell
.
├── _demos              # demo
│   └── ...
├── attr.go
├── cell.go
├── charset_stub.go
├── charset_unix.go
├── charset_windows.go
├── color.go
├── colorfit.go
├── console_stub.go
├── console_win.go
├── doc.go
├── encoding            # 编码
│   └── all.go
├── encoding.go
├── errors.go
├── event.go
├── interrupt.go
├── key.go              # 键盘事件定义
├── mouse.go            # 鼠标事件定义
├── nonblock_bsd.go
├── nonblock_unix.go
├── paste.go
├── resize.go
├── runes.go            # 字符映射定义(不能恰当转换的时候用到)
├── screen.go           # screen 接口定义
├── simulation.go
├── stdin_unix.go
├── style.go
├── termbox             # termbox 兼容实例
│   └── compat.go
├── terminfo            # 适配各种终端
│   └── ...
├── terms_default.go
├── terms_dynamic.go
├── terms_static.go
├── tscreen.go          # 实际使用的 screen. 重要
├── tscreen_stub.go
├── tscreen_unix.go
├── tty.go              # 终端操作封装
├── tty_unix.go
└── views               # 对 tcell 封装
    ├── _demos
    │   └── widget.go
    ├── app.go
    ├── boxlayout.go
    ├── cellarea.go
    ├── constants.go
    ├── panel.go
    ├── spacer.go
    ├── sstext.go
    ├── sstextbar.go
    ├── text.go
    ├── textarea.go
    ├── textbar.go
    ├── view.go
    └── widget.go
```

## 入口

views/app.go 中 run 方法

```go
func (app *Application) run() {

	screen := app.screen
	widget := app.widget

	if widget == nil {
		app.wg.Done()
		return
	}
	if screen == nil {
        // app 初始化
		if e := app.initialize(); e != nil {
			app.wg.Done()
			return
		}
		screen = app.screen
	}
	defer func() {
		screen.Fini()
		app.wg.Done()
	}()
    // screen 初始化
	screen.Init()
	screen.Clear()
    // widget 设置 view
	widget.SetView(screen)

loop:
	for {
		if widget = app.widget; widget == nil {
			break
		}
        // draw 和 show
		widget.Draw()
		screen.Show()

        // screen pollEvent
		ev := screen.PollEvent()
		switch nev := ev.(type) {
		case *eventAppQuit:
			break loop
		case *eventAppUpdate:
			screen.Show()
		case *eventAppRefresh:
			screen.Sync()
		case *eventAppFunc:
			nev.fn()
		case *tcell.EventResize:
			screen.Sync()
			widget.Resize()
		default:
			widget.HandleEvent(ev)
		}
	}
}
```

可以看到有几件事情
- app 初始化
- screen 初始化
- widget 设置 view
- 循环
  - draw 和 show
  - poollEvent
  - 处理事件

##

views/app.go 中 app.initialize 会调 tscreen.go 下面的方法

```go
func NewTerminfoScreenFromTty(tty Tty) (Screen, error) {
    // 从系统 env 中获取 TERM，然后找到 terminfo
	ti, e := terminfo.LookupTerminfo(os.Getenv("TERM"))
	if e != nil {
		ti, e = loadDynamicTerminfo(os.Getenv("TERM"))
		if e != nil {
			return nil, e
		}
		terminfo.AddTerminfo(ti)
	}
	t := &tScreen{ti: ti, tty: tty}

	t.keyexist = make(map[Key]bool)
	t.keycodes = make(map[string]*tKeyCode)
	if len(ti.Mouse) > 0 {
		t.mouse = []byte(ti.Mouse)
	}
	t.prepareKeys()
	t.buildAcsMap()
	t.resizeQ = make(chan bool, 1)                      // resize chan
	t.fallback = make(map[rune]string)
    // 加载字符映射
	for k, v := range RuneFallbacks {
		t.fallback[k] = v
	}

	return t, nil
}
```

可以看到有几件事情
- 从 env 中获取 TERM 环境变量，然后找 terminfo
- 创建 tScreen 实例，并初始化基本字段

## 

再来看 tscreen.go Init 方法

```go
func (t *tScreen) Init() error {
    // 初始化t.tty
	if e := t.initialize(); e != nil {
		return e
	}

	t.evch = make(chan Event, 10)                       // 事件chan
	t.keychan = make(chan []byte, 10)                   // 按键chan
	t.keytimer = time.NewTimer(time.Millisecond * 50)   // 处理按键的定时器
	t.charset = "UTF-8"

	t.charset = getCharset()
	if enc := GetEncoding(t.charset); enc != nil {
		t.encoder = enc.NewEncoder()
		t.decoder = enc.NewDecoder()
	} else {
		return ErrNoCharset
	}
	ti := t.ti

    ...

	t.colors = make(map[Color]Color)
	t.palette = make([]Color, t.nColors())
	for i := 0; i < t.nColors(); i++ {
		t.palette[i] = Color(i) | ColorValid
		// identity map for our builtin colors
		t.colors[Color(i)|ColorValid] = Color(i) | ColorValid
	}

	t.quit = make(chan struct{})                        // quit chan

	t.Lock()
	t.cx = -1
	t.cy = -1
	t.style = StyleDefault
	t.cells.Resize(w, h)
	t.cursorx = -1
	t.cursory = -1
	t.resize()
	t.Unlock()

	if err := t.engage(); err != nil {                  // 终端操作调用
		return err
	}

	return nil
}
```

这里干了几件事情
- 初始化 tty
- 事件、按键chan的初始化，按键定时器
- quit chan初始化
- 

### 初始化tty

```go
// NewDevTtyFromDev opens a tty device given a path.  This can be useful to bind to other nodes.
func NewDevTtyFromDev(dev string) (Tty, error) {
	tty := &devTty{
		dev: dev,
		sig: make(chan os.Signal),                                  // 接收窗口大小改变信号
	}
	var err error
	if tty.of, err = os.OpenFile(dev, os.O_RDWR, 0); err != nil {   // 拿到 /dev/tty 的 fd
		return nil, err
	}
	tty.fd = int(tty.of.Fd())
	if !term.IsTerminal(tty.fd) {
		_ = tty.f.Close()
		return nil, errors.New("not a terminal")
	}
	if tty.saved, err = term.GetState(tty.fd); err != nil {
		_ = tty.f.Close()
		return nil, fmt.Errorf("failed to get state: %w", err)
	}
	return tty, nil
}
```

tty 的初始化有个窗口大小改变的信号处理，后面会用到。另外一个是拿到 terminal 的 fd，已经初始状态，以便 app 退出的时候恢复

### 终端操作调用

```go
func (t *tScreen) engage() error {
	t.Lock()
	defer t.Unlock()
	if t.tty == nil {
		return ErrNoScreen
	}
	t.tty.NotifyResize(func() {             // 注册窗口大小改变回调函数
		select {
		case t.resizeQ <- true:             // 往 resizeQ 塞数据
		default:
		}
	})
	if t.running {
		return errors.New("already engaged")
	}
	if err := t.tty.Start(); err != nil {   // tty.Start
		return err
	}
	t.running = true
	if w, h, err := t.tty.WindowSize(); err == nil && w != 0 && h != 0 {
		t.cells.Resize(w, h)
	}
	stopQ := make(chan struct{})            // stop chan
	t.stopQ = stopQ
	t.enableMouse(t.mouseFlags)
	t.enablePasting(t.pasteEnabled)

	ti := t.ti
	t.TPuts(ti.EnterCA)
	t.TPuts(ti.EnterKeypad)
	t.TPuts(ti.HideCursor)
	t.TPuts(ti.EnableAcs)
	t.TPuts(ti.Clear)

	t.wg.Add(2)
	go t.inputLoop(stopQ)                   // 输入循环
	go t.mainLoop(stopQ)                    // main循环
	return nil
}
```

干了几件事情
- 注册窗口大小改变回调函数
  - 函数注册到了 tty.cb 中
- tty.Start
- stop chan 初始化
- 输入处理循环
- main循环
