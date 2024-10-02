# Golang-UIAutomator2
[![go doc](https://godoc.org/github.com/electricbubble/guia2?status.svg)](https://pkg.go.dev/github.com/electricbubble/guia2?tab=doc)
[![license](https://img.shields.io/github/license/electricbubble/guia2)](https://github.com/electricbubble/guia2/blob/master/LICENSE)

Client library implemented in Golang for [appium/appium-uiautomator2-server](https://github.com/appium/appium-uiautomator2-server)

## Extensions

- [electricbubble/guia2-ext-opencv](https://github.com/electricbubble/guia2-ext-opencv) Perform operations directly by specifying images

> If using `IOS` devices, check out [electricbubble/gwda](https://github.com/electricbubble/gwda)

## Installation
```bash
go get github.com/electricbubble/guia2
```

## Usage

> For the first use, you need to install two `apk` files on the `Android` device  
> `appium-uiautomator2-server-debug-androidTest.apk`  
> `appium-uiautomator2-server-vXX.XX.XX.apk`
>
>> You can choose to build the `apk` from [appium/appium-uiautomator2-server](https://github.com/appium/appium-uiautomator2-server#building-project)  
>> Or download directly from here [electricbubble/appium-uiautomator2-server-apk](https://github.com/electricbubble/appium-uiautomator2-server-apk/releases)
>  
>
> Then start `appium-uiautomator2-server` via `adb`  
> ```shell script
> adb shell am instrument -w io.appium.uiautomator2.server.test/androidx.test.runner.AndroidJUnitRunner
> # ⬇️ Run in the background
> adb shell "nohup am instrument -w io.appium.uiautomator2.server.test/androidx.test.runner.AndroidJUnitRunner >/sdcard/uia2server.log 2>&1 &"
> # or
> adb -s $serial shell "nohup am instrument -w io.appium.uiautomator2.server.test/androidx.test.runner.AndroidJUnitRunner >/sdcard/uia2server.log 2>&1 &"
> ```

### `guia2.NewUSBDriver()`
During the use of this function, the `Android` device must always maintain a `USB` connection (this function is also used for `emulators`)

### `guia2.NewWiFiDriver("192.168.1.28")`
1. First connect the `Android` device via `USB`
2. Let the device listen for TCP/IP connections on port 5555
	```shell script
	adb tcpip 5555
   # or
	adb -s $serial tcpip 5555
	```
3. Find the `IP` of the `Android` device (you can choose to disconnect the `USB` connection from this step)
4. Connect to the `Android` device via `IP`
	```shell script
	adb connect $deviceIP
	```
5. Confirm the connection status
	```shell script
	adb devices
	```
	If you see a device in the following format, the connection is successful
	```shell script
	$deviceIP:5555    device
	```

```go
package main

import (
	"fmt"
	"github.com/electricbubble/guia2"
	"io/ioutil"
	"log"
	"os"
)

func main() {
	driver, err := guia2.NewUSBDriver()
	// driver, err := guia2.NewWiFiDriver("192.168.1.28")
	checkErr(err)
	defer func() { _ = driver.Dispose() }()

	// err = driver.AppLaunch("tv.danmaku.bili")
	err = driver.AppLaunch("tv.danmaku.bili", guia2.BySelector{ResourceIdID: "tv.danmaku.bili:id/action_bar_root"})
	checkErr(err, "launch the app until the element appears")

	// fmt.Println(driver.Source())
	// return

	deviceSize, err := driver.DeviceSize()
	checkErr(err)

	var startX, startY, endX, endY int
	startX = deviceSize.Width / 2
	startY = deviceSize.Height / 2
	endX = startX
	endY = startY / 2
	err = driver.Swipe(startX, startY, endX, endY)
	checkErr(err)

	var startPoint, endPoint guia2.PointF
	startPoint = guia2.PointF{X: float64(startX), Y: float64(startY)}
	endPoint = guia2.PointF{X: startPoint.X, Y: startPoint.Y * 1.6}
	err = driver.SwipePointF(startPoint, endPoint)
	checkErr(err)

	element, err := driver.FindElement(guia2.BySelector{ResourceIdID: "tv.danmaku.bili:id/expand_search"})
	checkErr(err)

	err = element.Click()
	checkErr(err)

	bySelector := guia2.BySelector{UiAutomator: guia2.NewUiSelectorHelper().Focused(true).String()}
	element, err = waitForElement(driver, bySelector)
	checkErr(err)

	err = element.SendKeys("Fog Hill of Five Elements")
	checkErr(err)

	err = driver.PressKeyCode(guia2.KCEnter, guia2.KMEmpty)
	checkErr(err)

	bySelector = guia2.BySelector{UiAutomator: guia2.NewUiSelectorHelper().TextStartsWith("Anime").String()}
	element, err = waitForElement(driver, bySelector)
	checkErr(err)
	checkErr(element.Click())

	bySelector = guia2.BySelector{UiAutomator: guia2.NewUiSelectorHelper().Text("Watch Now").String()}
	element, err = waitForElement(driver, bySelector)
	checkErr(err)
	checkErr(element.Click())

	bySelector = guia2.BySelector{ResourceIdID: "tv.danmaku.bili:id/videoview_container_space"}
	element, err = waitForElement(driver, bySelector)
	checkErr(err)

	// time.Sleep(time.Second * 5)

	screenshot, err := element.Screenshot()
	checkErr(err)
	userHomeDir, _ := os.UserHomeDir()
	checkErr(ioutil.WriteFile(userHomeDir+"/Desktop/element.png", screenshot.Bytes(), 0600))

	err = driver.PressKeyCode(guia2.KCMediaPause, guia2.KMEmpty)
	checkErr(err)

	err = driver.PressBack()
	checkErr(err)
}

func waitForElement(driver *guia2.Driver, bySelector guia2.BySelector) (element *guia2.Element, err error) {
	var ce error
	exists := func(d *guia2.Driver) (bool, error) {
		element, ce = d.FindElement(bySelector)
		if ce == nil {
			return true, nil
		}
		// If you return an error directly, it will terminate `driver.Wait`
		return false, nil
	}
	if err = driver.Wait(exists); err != nil {
		return nil, fmt.Errorf("%s: %w", err.Error(), ce)
	}
	return
}

func checkErr(err error, msg ...string) {
	if err == nil {
		return
	}

	var output string
	if len(msg) != 0 {
		output = msg[0] + " "
	}
	output += err.Error()
	log.Fatalln(output)
}

```

> Thanks to the buddy who provided the `Redmi Note 5A`


![example](https://github.com/electricbubble/ImageHosting/blob/master/img/202008192034_guia2.gif)


## Thanks

Thank you [JetBrains](https://www.jetbrains.com/?from=gwda) for providing free open source licenses

