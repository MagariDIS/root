# 最近の検証


---
# ESP8266 Linknode R4 で REST Apiによる 電源タップ制御アプリを作成する手順
### AS OF 2020/10/24

LinknodeR4をWifiWebサーバーとしてプログラムし、簡単なREST ApiでhttpGETおよびPOST要求に応答する物です。

ESP8266でWiFiを使うときに、SSIDをコード内に埋め込まなくていいようにするWiFiManagerを使用しています

## 1.　LinknodeR4　とは？

LinkNode R4はWiFiリレーコントローラーで、Arduinoプログラミングに対応したESP-12f ESP8266 WiFiモジュールで動作します。 

```
ESP-12f ESP8266 WiFi moduleのスペック

4 Channel リレーは 277V AC， 10A　又は　125V AC， 12A　まで対応

5V DC power　端子にUSBコネクターを増設しています。

ジャンパーPin 左側1-2番短絡で Program via UART　モード
ジャンパーPin 右側2-3番短絡で Boot from flash　モード
4 indiator LEDs　でリレーのON/OFF状態が表示れます
```

4つのリレーチャンネルがあり、各チャンネルはオンボードリレー経由でハイパワーデバイス(最大10A)を制御できます。

詳しくはAmazonで
https://www.amazon.co.jp/LinkSprite-211201004-LinkNode-WiFiリレーコントローラー-4チャンネルリレーモジュール


## 2. 初期セットアップ
電源オンを30秒程度でWi-Fiルーターの他に"AutoConnectAP"というAPが見つかります。

これがLinknode R4（ESP8266）なので、これを選択します。
ssidのパスワードは　password です。

「Configure WiFi」をタップすると、近くにあるWi-Fi APのリストが表示されますので
利用可能なAPを選択しパスワードを設定すると、自動的に再始動してますので
以後は http://LinknodeR4.local:80/　で接続可能になります 


## 3. 使い方

使用するGPIOポートはソースにハードコートしています

以下例では

```
int relays[] = { 14, 12, 13, 16 };  // GPIO pins for each relay
```
 GPIO  | relay num | LED  | 端子位置 
:------:|:---------:|-------|:---------------------
GPIO14 | 0番 | D4　LED | 左上のリレー
GPIO12 | 1番 | D10　LED | 右上のリレー
GPIO13 | 2番 | D8　LED | 左下のリレー
GPIO16 | 3番 | D3　LED | 右下のリレー


となります。

### REST 呼び出しの例


#### 1. 4番目の端子をON
~~~
http://linknoder4.local/api/relay/3/on　　
~~~

#### 2. 1番目の端子をOFF
~~~
http://linknoder4.local/api/relay/0/off　　
~~~

#### 3. 全ての端子をON
~~~
http://linknoder4.local/api/relay/all/on　　
~~~

#### 4. リレーの現状確認
~~~
http://linknoder4.local/api/relay/　
~~~


## 4. ソースコード 

```ino

/********************************************
 * 
 *  LinknodeR4 WifiWebサーバー
 * 
 * http postリクエストを実行してリレーをアクティブにし、
 * GETリクエストを実行してステータスを取得します。
 * もローカルネットワークに接続できます。
 * 
 * ##API Documentation
 * 
 * ## Wifi設定 をresetする
 * http://sample.local/cmd?wifi=reset
 * 
 * ## 特定のリレーをONする　POST api/relay/<relay#>/on
 * Response: 200 { "message": "Relay <relay#> ON" }
 * 
 * ##特定のリレーをOFFする　 POST api/relay/<relay#>/off
 * Response: 200 { "message": "Relay <relay#> OFF" }
 * 
 * ##全てのリレーをONする POST api/relay/all/on
 * Response: 200 { "message": "All Relays = ON" }
 * 
 * ##全てのリレーをOFFする POST api/relay/all/off 
 * Response: 200 { "message": "All Relays = OFF" }
 * 
 * ##すべてのリレー状態を得る GET api/relay
 * Response: 200 { "relays": [ state0, state1, state2, state3 ] } 
 *
 * 
 * ##誤った操作 GET api/relay
 * Response: 400 - Bad Request. Response body: { "message": "Invalid request" }
 * 
 * ******************************************/

#include <FS.h>
#include <ESP8266WiFi.h>
#include <WiFiClient.h>
#include <ESP8266WebServer.h>
#include <ESP8266mDNS.h>

#include <WiFiManager.h> 


ESP8266WebServer server(80);
static const char* cpResponse200 = "<HTML>"
 "<BODY style='background-color:#ffffde;font-family:sans-serif;font-size:40px;'>"
 "CONTROL WEB<br/><br/>"
 "<a href=/cmd?CMD=dummy>DUMMY</a><br/>"
 "</BODY></HTML>\r\n";
 
boolean deviceStates[] = { false, false, false, false };  // Initial state of each relay
int relays[] = { 14, 12, 13, 16 };  // GPIO pins for each relay

void setup(void){

  Serial.begin(115200);
  Serial.println("");

 //WiFiManager
  WiFiManager wifiManager;
  wifiManager.setBreakAfterConfig(true);
  if (!wifiManager.autoConnect("AutoConnectAP", "password")) {
    Serial.println("failed to connect, we should reset as see if it connects");
    delay(3000);
    ESP.reset();
    delay(5000);
  }
  
  Serial.print("Connected to ");
  Serial.println(WiFi.SSID());
  
  Serial.print("IP address: ");
  Serial.println(WiFi.localIP());

  Serial.print("MAC address: ");
  byte mac[6];
  WiFi.macAddress(mac);
  Serial.println(getMacString(mac));
  
  Serial.print("Gateway: ");
  Serial.println(WiFi.gatewayIP());

  // Set up mDNS responder:
  if (!MDNS.begin("linknoder4")) { // linknoder4.local
    Serial.println("Error setting up MDNS responder!");
    while (1) {
      delay(1000);
    }
  }
  Serial.println("mDNS responder started");
  Serial.println("");

  // WebServer
  server.begin();

  // Add service to MDNS-SD
  MDNS.addService("http", "tcp", 80);

  server.on("/cmd", ctlCommand);

  server.on("/api/relay", handleRoot);

  Serial.println("Initialize relays...");
  for (byte relay = 0; relay < sizeof(relays) / sizeof(int); relay++)  {
    
    pinMode(relays[relay], OUTPUT);
    digitalWrite(relays[relay], deviceStates[relay] ? HIGH : LOW);

    String request = "/api/relay/" + String(relay) + "/";
    Serial.println("Relay " + String(relay) + " url: " + request);
    server.on((request + "on").c_str(), [relay](){
      setState(relay, true);
    });
    server.on((request + "off").c_str(), [relay](){
      setState(relay, false);
    });
  }
  server.on("/api/relay/all/on", [](){
    setAll(true);
  });

  server.on("/api/relay/all/off", [](){
    setAll(false);
  });

  server.onNotFound(handleNotFound);
  server.begin();
  
  Serial.println("HTTP server started");
}



void loop(void){
  MDNS.update();
  server.handleClient();
}

// This method is called when a request is made to the root of the server. i.e. http://myserver
void handleRoot() {
  String response = "{ \"relays\": [ ";
  String seperator = "";
  for (int relay = 0; relay < sizeof(relays) / sizeof(int); relay++)  {
    response += seperator + String(deviceStates[relay]);
    seperator = ", ";
  }
  response += " ] }";
  server.send(200, "application/json", response);
}

void ctlCommand() {
  String cmd = server.arg("wifi");
  if (cmd == "dummy")  {
    // hogehoge...
  }
  else   if (cmd == "reset")  { //　resetコマンドでWiFi再設定
    WiFiManager wifiManager;
    delay(1000);
    Serial.println("");
    Serial.println("WiFi Reset !");
    Serial.println("");
    wifiManager.resetSettings();
    delay(3000);
    ESP.reset();
  }
  server.send(200, "text/html", cpResponse200);
}


// This method is called when an undefined url is specified by the caller
void handleNotFound() {
  server.send(400, "text/plain", "{ \"message\": \"Invalid request\" }");
}

void setAll(boolean state){
  for (byte relay = 0; relay < sizeof(relays) / sizeof(int); relay++)  {
    digitalWrite(relays[relay], state ? HIGH : LOW);
    deviceStates[relay] = state;
  }
  String response = "All Relays = " + String(state ? "ON" : "OFF");
  server.send(200, "application/json", "{ \"message\": \"" + response + "\" }");
}

void setState(int relay, boolean state) {
    digitalWrite(relays[relay], state ? HIGH : LOW);
    deviceStates[relay] = state;
    String response = "Relay " + String(relay) + " " + (state ? "ON" : "OFF");
    server.send(200, "application/json", "{ \"message\": \"" + response + "\" }");
    Serial.println(response);
}

String getMacString(byte *mac) {
  String result = "";
  String seperator = "";
  for(int b = 0; b < 6; b++) {
    result += seperator + String(mac[b], HEX);
    seperator = ":";
  }
  return result;
}
```




---
# Rasberry PI 上でJavaJDK ＋ OpenCV 4.0 による背景ノイズ除去アプリを作成する手順
### AS OF 2020/10/17 



OCRには適さない 背景ありの暗い画像

![元画像](https://github.com/MagariDIS/root/blob/images/cleanOcr_imput.jpg)


文字らしき物の輪郭を抽出しバックグラウンドを削除します
![クリーニング画像](https://github.com/MagariDIS/root/blob/images/cleanOcr_output.jpg)


## 環境構築

1. hw : Raspberry pi 3
2. java : openjdk-11-jdk-headless



## Install

### 1. JavaJDK（開発環境）の導入と OpenCVソースコードのダウンロード

```sh

sudo apt install default-jdk

cd /usr/local/src

git clone https://github.com/opencv/opencv.git
cd opencv
git checkout 4.5.0

git clone https://github.com/opencv/opencv_contrib.git

cd opencv_contrib
git checkout 4.5.0
```

### 2. コンパイル時のメモリー対策としてスワップファイルサイズを2Gへ変更

```sh

swapon
sudo sed -i -e 's/SWAPSIZE=.*/SWAPSIZE=2048/g' /etc/dphys-swapfile
sudo systemctl restart dphys-swapfile.service
swapon
```

## 3. OpenCVのビルド

```sh

mkdir -p /usr/local/src/opencv/build && cd /usr/local/src/opencv/build

export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-armhf
cmake -DBUILD_SHARED_LIBS=OFF -DWITH_FFMPEG=ON -DOPENCV_EXTRA_MODULES_PATH=/usr/local/src/opencv_contrib/modules /usr/local/src/opencv ..

make -j2
sudo make install
sudo ldconfig
```

### 3.1 Javaラッパーをシンボリックリンクする

```sh

cd /usr/java/packages/lib
ln -s libopencv_java450.so -> /usr/local/share/java/opencv4/libopencv_java450.so
```

## 4. ソースコード cleanOcr.java

openCV 4.0 対応するように変更しました
<a href="https://www.it-swarm-ja.tech/ja/java/%E7%94%BB%E5%83%8F%E3%81%8B%E3%82%89%E8%83%8C%E6%99%AF%E3%83%8E%E3%82%A4%E3%82%BA%E3%82%92%E9%99%A4%E5%8E%BB%E3%81%97%E3%81%A6%E3%80%81ocr%E3%81%AE%E3%83%86%E3%82%AD%E3%82%B9%E3%83%88%E3%82%92%E3%82%88%E3%82%8A%E6%98%8E%E7%A2%BA%E3%81%AB%E3%81%97%E3%81%BE%E3%81%99/1056484951/">オリジナルコードの所在</a>


```java

import java.util.ArrayList;
import java.util.List;

import org.opencv.core.*;
import org.opencv.imgcodecs.Imgcodecs;
import org.opencv.imgproc.Imgproc;

public class cleanOcr {

    public static void main( String args[] ) {
	System.loadLibrary( Core.NATIVE_LIBRARY_NAME );

	// Mat im = Highgui.imread("imput.png", 0);
	Mat im = Imgcodecs.imread("imput.jpg", 0);
	// apply Otsu threshold
	Mat bw = new Mat(im.size(), CvType.CV_8U);
	Imgproc.threshold(im, bw, 0, 255, Imgproc.THRESH_BINARY_INV | Imgproc.THRESH_OTSU);
	// take the distance transform
	Mat dist = new Mat(im.size(), CvType.CV_32F);
	Imgproc.distanceTransform(bw, dist, Imgproc.CV_DIST_L2, Imgproc.CV_DIST_MASK_PRECISE);
	// threshold the distance transform
	Mat dibw32f = new Mat(im.size(), CvType.CV_32F);
	final double SWTHRESH = 8.0;    // stroke width threshold
	Imgproc.threshold(dist, dibw32f, SWTHRESH/2.0, 255, Imgproc.THRESH_BINARY);
	Mat dibw8u = new Mat(im.size(), CvType.CV_8U);
	dibw32f.convertTo(dibw8u, CvType.CV_8U);
	
	Mat kernel = Imgproc.getStructuringElement(Imgproc.MORPH_RECT, new Size(3, 3));
	// open to remove connections to stray elements
	Mat cont = new Mat(im.size(), CvType.CV_8U);
	Imgproc.morphologyEx(dibw8u, cont, Imgproc.MORPH_OPEN, kernel);
	// find contours and filter based on bounding-box height
	final double HTHRESH = im.rows() * 0.5; // bounding-box height threshold
	List<MatOfPoint> contours = new ArrayList<MatOfPoint>();
	List<Point> digits = new ArrayList<Point>();    // contours of the possible digits
	Imgproc.findContours(cont, contours, new Mat(), Imgproc.RETR_CCOMP, Imgproc.CHAIN_APPROX_SIMPLE);
	for (int i = 0; i < contours.size(); i++)
	{
    	if (Imgproc.boundingRect(contours.get(i)).height > HTHRESH)
    	{
        	// this contour passed the bounding-box height threshold. add it to digits
        	digits.addAll(contours.get(i).toList());
    	}   
	}
	// find the convexhull of the digit contours
	MatOfInt digitsHullIdx = new MatOfInt();
	MatOfPoint hullPoints = new MatOfPoint();
	hullPoints.fromList(digits);
	Imgproc.convexHull(hullPoints, digitsHullIdx);
	// convert hull index to hull points
	List<Point> digitsHullPointsList = new ArrayList<Point>();
	List<Point> points = hullPoints.toList();
	for (Integer i: digitsHullIdx.toList())
	{
    		digitsHullPointsList.add(points.get(i));
	}
	MatOfPoint digitsHullPoints = new MatOfPoint();
	digitsHullPoints.fromList(digitsHullPointsList);
	// create the mask for digits
	List<MatOfPoint> digitRegions = new ArrayList<MatOfPoint>();
	digitRegions.add(digitsHullPoints);
	Mat digitsMask = Mat.zeros(im.size(), CvType.CV_8U);
	Imgproc.drawContours(digitsMask, digitRegions, 0, new Scalar(255, 255, 255), -1);
	// dilate the mask to capture any info we lost in earlier opening
	Imgproc.morphologyEx(digitsMask, digitsMask, Imgproc.MORPH_DILATE, kernel);
	// cleaned image ready for OCR
	Mat cleaned = Mat.zeros(im.size(), CvType.CV_8U);
	dibw8u.copyTo(cleaned, digitsMask);
	// feed cleaned to Tesseract
	Imgcodecs.imwrite("output.jpg", cleaned);
    	System.out.println("cleaning sucess!");
    }
}

```

## 5. コンパイルと実行

```sh

javac -classpath /usr/local/share/java/opencv4/opencv-450.jar cleanOcr.java

java -classpath .:/usr/local/share/java/opencv4/opencv-450.jar cleanOcr
```

* 実行時にクラスパスを指定するが、カレントディレクトリも指定していないとMainが見つからないと言われるので注意



### 5.1 tesseract で OCR してみる

tesseract output.jpg output

```
Tesseract Open Source OCR Engine v4.0.0 with Leptonica
Warning: Invalid resolution 0 dpi. Using 70 instead.
Estimating resolution as 1547
```

cat output.txt 
```
NIA s 9:31 4
NTSOI3SGR

PB! 7OB02 4
```

結果は微妙ですが改善しています

---
