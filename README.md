# RegisterSystem
A hospital register system App has hover touch function by Mediapipe hands.
## 摘要
近幾年，由於疫情的影響，產生許多病毒傳播的途徑，接觸傳染也是其中之一，因此為
了降低醫療院所產生接觸傳染的風險，本專題將設計一個無接觸門診自助報到系統，並使用
MediaPipe hands 實現懸浮觸控。此外本專題將探討電容式懸浮觸控與影像辨識懸浮觸控的差
異與特色，及介紹 MediaPipe hands 的模型架構。透過使用影像辨識完成手部辨識，讓使用
者透過手勢操控無需接觸螢幕即可完成自助報到，從而降低病毒傳播風險。最後本專題提出
了此系統的精進方向和未來懸浮觸控的技術潛力。
## 前言  
### 研究背景及動機  
近年來在疫情的影響下公共場所成為病毒傳播的媒介，因此人們開始研究各種方法
來避免病毒擴散。例如：透過手部感應的自動酒精機、用手機確認附近商店口罩的存貨
量以避免排隊造成的群聚感染等，而在某些醫院門口，會放置一台自助報到機幫助病患
快速完成報到，但是在醫院這種傳染風險高的場所，大家都想避免和其他病患發生接
觸，尤其是報到機這種病患會經常接觸的機器，因此降低接觸感染的風險也成為醫療院
所的重要課題。由於上述原因，我們想設計一款無接觸自助掛號系統，解決觸碰螢幕產
生的接觸感染。  
### 研究目的  
為了解決觸碰螢幕產生的接觸感染，我們想到透過懸浮觸控的方式解決此問題。我們
運用 MediaPipe 透過影像辨識的方式實現懸浮觸控，製作出一款模擬醫療院所自助報到
機的 App，讓使用者無需接觸螢幕，即可完成自助報到。  
研究目地如下：  
(一) 透過懸浮觸控避免接觸感染  
(二) 透過 MediaPipe 完成手部辨識  
(三) 分析 MediaPipe 回傳手部座標，完成顯示螢幕游標及觸控偵測  
(四) 完成門診報到系統  
## MediaPipe Hands  
手部偵測在許多技術領域中是一項提升使用者體驗的關鍵部分，例如：手語辨識、手勢控制等、虛擬實境等 (Bazarevsky et al., 2019)，因此MediaPipe Hands 透過機器學習提供可實時偵測手部的影像辨識系統。此系統主要由兩個模型組成：手掌偵測模型與手部座標模型。  
### 手掌偵測模型(Palm Detection Model)
手掌偵測模型主要運作在整張影像中，負責偵測手掌，並將偵測到的手部框出回傳給手部座標模型  (Bazarevsky et al., 2019)，此外為了提升模型的運作效率，手掌偵測模型僅會在影像的第一偵畫面，或影像中失去手掌時運作 (Fan Zhang et al., 2020)。  
### 手部座標模型(Hand Landmark Model)
在運作手掌偵測模型之後，手部座標模型將接收手掌偵測模型傳入的手部影像，經過真實世界影像與合成影像訓練後，產生三樣輸出，包括21個手部座標，如圖四、一個表示手部是否存在於影像中的數值，及表示左手或右手的數值 (Fan Zhang et al., 2020)。  
### 懸浮觸控程式碼：
```java
private Hands hands; //懸浮觸控手部物件
private static final boolean RUN_ON_GPU = true;//是否使用GPU偵測手部
private CameraInput cameraInput;//相機物件
private SolutionGlSurfaceView<HandsResult> glSurfaceView;
private LandmarkProto.NormalizedLandmark wristLandmark;//手部座標物件
private MotionEvent clickEvent;//觸發螢幕觸控物件
private float handX, handY, handZ, fingerX, fingerY, fingerZ, fingerBtmX, fingerBtmY; //手掌座標、指尖座標、手指根部座標
private int parentX, parentY;
   //現態指尖、指跟距離；前一時刻指尖、指跟距離；現態手掌距離、前一時刻手掌距離(觸控偵測用)
private double nowFingerDistance, beforeFingerDistance, beforeHandX, beforeHandY, nowHandX, nowHandY;
private boolean isActionDown, isMoving, isFingerMoving;//是否點擊螢幕、是否正在移動手部、是否滑動螢幕
private int cursorX, cursorY;//游標座標
private float touchX, touchY;//觸控座標

private void setCamera() {
hands.setResultListener(new ResultListener<HandsResult>() {//取得回傳手部座標
   @Override
   public void run(HandsResult result) {
       if(!result.multiHandLandmarks().isEmpty()) {//若已偵測到手部
           wristLandmark = result.multiHandLandmarks().get(0).getLandmarkList().get(HandLandmark.MIDDLE_FINGER_MCP);
           handX = wristLandmark.getX() * 100;//取手掌座標
           handY = wristLandmark.getY() * 100;
           handZ = wristLandmark.getZ() * 100;

           wristLandmark = result.multiHandLandmarks().get(0).getLandmarkList().get(HandLandmark.INDEX_FINGER_TIP);
           fingerX = wristLandmark.getX() * 100;//取指尖座標
           fingerY = wristLandmark.getY() * 100;
           fingerZ = wristLandmark.getZ() * 100;

           wristLandmark = result.multiHandLandmarks().get(0).getLandmarkList().get(HandLandmark.INDEX_FINGER_MCP);
           fingerBtmX = wristLandmark.getX() * 100;//取指跟座標
           fingerBtmY = wristLandmark.getY() * 100;
           setCursor();//顯示游標
           sendFingerPos();//觸控偵測
       } else {
           if(isActionDown) { //未偵測到手部且已觸發點擊螢幕動作->觸發鬆開螢幕
               isActionDown = false;
               clickEvent = MotionEvent.obtain(SystemClock.uptimeMillis(), SystemClock.uptimeMillis(),
                       MotionEvent.ACTION_UP, 0, 0, 0);
               setEvent(clickEvent);//觸發鬆開螢幕
           }
           if(cursor.getParent() != null) { //為偵測到手部則清除游標icon
               windowManager.removeView(cursor);//清除游標icon
           }
       }
       glSurfaceView.setRenderData(result);
       glSurfaceView.requestRender();
   }
});
  }

private void setCursor() {
   if(isFingerMoving){ //若正在進行觸控動作則停止移動游標
       return;
   }
   cursorX = (int) (((handX - 40) / 35) * parentX); //set cursor position
   cursorY = (int) (((handY - 80) / 25) * parentY);
   cursorLayout.x = cursorX;
   cursorLayout.y = cursorY;
   runOnUiThread(new Runnable() {
       @Override
       public void run() {
           if(cursor.getParent() == null) {  //update cursor view
               windowManager.addView(cursor, cursorLayout);
           }
           windowManager.updateViewLayout(cursor, cursorLayout);
       }
   });
}

private void sendFingerPos() {
   beforeFingerDistance = nowFingerDistance; //update finger position
   beforeHandX = nowHandX;//update hand position
   nowHandX = handX;
   beforeHandY = nowHandY;
   nowHandY = handY;

   touchX = cursorX + (parentX / 2);//set touch position
   touchY = cursorY + (parentY / 2);
nowFingerDistance = Math.sqrt(Math.pow(fingerBtmX - fingerX, 2) + Math.pow(fingerBtmY - fingerY, 2));//計算指尖與指跟距離
   double different = nowFingerDistance - beforeFingerDistance;//計算手指距離變化量
   double handDistance = Math.sqrt(Math.pow(nowHandX - beforeHandX, 2) + Math.pow(nowHandY - beforeHandY, 2));//計算手掌距離變化量
   
   if(handDistance >= 0.85f) { //判斷手部是否正在移動
       isMoving = true;
       Log.v("handmov:", "move");
   } else if(isMoving){
       isMoving = false;
       Log.v("handmov:", "not move");
   }

   if(different <= -1.5){//判斷手指是否正在移動
       isFingerMoving = true;
   }else{
       isFingerMoving = false;
   }

   if(different <= -1.2f && !isActionDown && !isMoving) { //若手指距離變化量
<-1.2且手部停止移動且無觸發點擊螢幕，則觸發點擊螢幕

       isActionDown = true;
       clickEvent = MotionEvent.obtain(SystemClock.uptimeMillis(), SystemClock.uptimeMillis(),
               MotionEvent.ACTION_DOWN, touchX, touchY, 0);
       setEvent(clickEvent);//觸發觸控動作
   }

   if(isActionDown) {//若已觸發點擊螢幕，則觸發滑動螢幕
       clickEvent = MotionEvent.obtain(SystemClock.uptimeMillis(), SystemClock.uptimeMillis(),
               MotionEvent.ACTION_MOVE, touchX, touchY, 0);
       setEvent(clickEvent);//觸發觸控動
   }

   if(different >= 0.9f && isActionDown) {//若手指距離變化量>0.9，且已觸發點擊螢幕，則觸發鬆開螢幕並設置點擊螢幕為false
       isActionDown = false;
       clickEvent = MotionEvent.obtain(SystemClock.uptimeMillis(), SystemClock.uptimeMillis(),
               MotionEvent.ACTION_UP, touchX, touchY, 0);
       setEvent(clickEvent);//觸發觸控動作
   }
}
private void setEvent(MotionEvent event) { //send touch event to different screen layer
   runOnUiThread(new Runnable() {
       @Override
       public void run() {//若第二層對話框正在顯示，則對第二層對話框觸發觸控動作
           if(secondDialog != null && secondDialog.isShowing() && mainDialog != null && mainDialog.isShowing()) {
               event.setLocation(touchX - ((mainDialog.getWindow().getDecorView().getWidth() - secondDialog.getWindow().getDecorView().getWidth()) / 2),
                       touchY - ((parentY - secondDialog.getWindow().getDecorView().getHeight()) / 2));
               secondDialog.dispatchTouchEvent(event);//觸發觸控
           } else if(mainDialog != null && mainDialog.isShowing()) {
               event.setLocation(touchX, touchY - ((parentY - mainDialog.getWindow().getDecorView().getHeight()) / 2));//若第一層對話框正在顯示，則對第一層對話框觸發觸控動作
               mainDialog.dispatchTouchEvent(event);
           } else {
               dispatchTouchEvent(event); //若兩層對話框皆未顯示，則對主螢幕觸發觸控動作
           }
           Log.v("touch","" + touchX + " "+ touchY);
       }
   });
```


