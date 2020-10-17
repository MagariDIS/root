# root
Just a Test Repository

---
# OpenCVによる背景ノイズ除去をJavaで作成する手順

OCRには適さない 背景ありの暗い画像

![元画像](https://github.com/MagariDIS/root/blob/images/cleanOcr_imput.jpg)


文字らしき物の輪郭を抽出しバックグラウンドを削除します
![クリーニング画像](https://github.com/MagariDIS/root/blob/images/cleanOcr_output.jpg)


## 環境構築

1. hw : Raspberry pi 3
2. java : openjdk-11-jdk-headless



## Install

### 1. JavaJDK（開発環境）の導入と OpenCVソースコードのダウンロード

sudo apt install default-jdk

cd /usr/local/src

git clone https://github.com/opencv/opencv.git
cd opencv
git checkout 4.5.0

git clone https://github.com/opencv/opencv_contrib.git

cd opencv_contrib
git checkout 4.5.0

### 2. コンパイル時のメモリー対策としてスワップファイルサイズを2Gへ変更

swapon
sudo sed -i -e 's/SWAPSIZE=.*/SWAPSIZE=2048/g' /etc/dphys-swapfile
sudo systemctl restart dphys-swapfile.service
swapon


## 3. OpenCVのビルド

mkdir -p /usr/local/src/opencv/build && cd /usr/local/src/opencv/build

export JAVA_HOME=/usr/lib/jvm/java-11-openjdk-armhf
cmake -DBUILD_SHARED_LIBS=OFF -DWITH_FFMPEG=ON -DOPENCV_EXTRA_MODULES_PATH=/usr/local/src/opencv_contrib/modules /usr/local/src/opencv ..

make -j1
sudo make install
sudo ldconfig

## 3.2 Javaラッパーをシンボリックリンクする

cd /usr/java/packages/lib
ln -s libopencv_java450.so -> /usr/local/share/java/opencv4/libopencv_java450.so

## 4. ソースコード cleanOcr.java

```java

import java.util.ArrayList;
import java.util.List;

import org.opencv.core.*;
//import org.opencv.core.CvType;
//import org.opencv.core.Mat;
//import org.opencv.core.MatOfInt;
//import org.opencv.core.MatOfInt4;
//import org.opencv.core.MatOfPoint;
//import org.opencv.core.Point;
//import org.opencv.core.Scalar;
import org.opencv.imgcodecs.Imgcodecs;
import org.opencv.imgproc.Imgproc;

public class cleanOcr {

    public static void main( String args[] ) {
	System.loadLibrary( Core.NATIVE_LIBRARY_NAME );

	// https://www.it-swarm-ja.tech/ja/java/%E7%94%BB%E5%83%8F%E3%81%8B%E3%82%89%E8%83%8C%E6%99%AF%E3%83%8E%E3%82%A4%E3%82%BA%E3%82%92%E9%99%A4%E5%8E%BB%E3%81%97%E3%81%A6%E3%80%81ocr%E3%81%AE%E3%83%86%E3%82%AD%E3%82%B9%E3%83%88%E3%82%92%E3%82%88%E3%82%8A%E6%98%8E%E7%A2%BA%E3%81%AB%E3%81%97%E3%81%BE%E3%81%99/1056484951/
       // https://stackoverflow.com/questions/33881175/remove-background-noise-from-image-to-make-text-more-clear-for-ocr/38266049
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
    	System.out.println("grabcut sucess!");
    }
}

```

## 5. コンパイルと実行

javac -classpath /usr/local/share/java/opencv4/opencv-450.jar cleanOcr.java

java -classpath .:/usr/local/share/java/opencv4/opencv-450.jar cleanOcr

*** 実行時にクラスパスを指定するが、カレントディレクトリも指定していないとMainが見つからないと言われるので注意 ***



## 5.1 tesseract で OCR してみる

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
