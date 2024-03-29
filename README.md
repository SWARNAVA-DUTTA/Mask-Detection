# MaskDetection
 
<div float:left>
 <img src="https://user-images.githubusercontent.com/46235752/156165647-84d3849a-eb16-4570-beb0-98ab78dd8b30.png" width="400" height="300">
 <img src="https://user-images.githubusercontent.com/46235752/156165380-f3f4be63-f291-4919-b6b7-5997aa7bd9f6.png" width="400" height="300">
</div>
<br/><br/>


1. Import the packages
   ```python
   from keras.applications.mobilenet_v2 import preprocess_input
   from keras.preprocessing.image import img_to_array
   from keras.models import load_model
   from imutils.video import VideoStream
   import numpy as np
   import imutils
   import cv2
   ```
2. Grab the dimensions of the frame and then construct a blob from it
   ```python
   def detect_and_predict_mask(frame, faceNet, maskNet):
   (h, w) = frame.shape[:2]
   blob = cv2.dnn.blobFromImage(frame, 1.0, (224, 224),
                                 (104.0, 177.0, 123.0))
   ```
3. Pass the blob through the network and obtain the face detections 
   ```python
   faceNet.setInput(blob)
   detections = faceNet.forward()
   print(detections.shape)
   ```
4. Initialize our list of faces, their corresponding locations, and the list of predictions from our face mask network 
   ```python
   faces = []
   locs = []
   preds = []
   ```
5. Loop over the detections
   ```python
   for i in range(0, detections.shape[2]):

    # extract the confidence (i.e., probability) associated with the detection
    confidence = detections[0, 0, i, 2]

    # filter out weak detections by ensuring the confidence is greater than the minimum confidence
    if confidence > 0.5:
        # compute the (x, y)-coordinates of the bounding box for
        # the object
        box = detections[0, 0, i, 3:7] * np.array([w, h, w, h])
        (startX, startY, endX, endY) = box.astype("int")

        # ensure the bounding boxes fall within the dimensions of the frame
        (startX, startY) = (max(0, startX), max(0, startY))
        (endX, endY) = (min(w - 1, endX), min(h - 1, endY))

        # extract the face ROI, convert it from BGR to RGB channel ordering, resize it to 224x224, and preprocess it
        face = frame[startY:endY, startX:endX]
        face = cv2.cvtColor(face, cv2.COLOR_BGR2RGB)
        face = cv2.resize(face, (224, 224))
        face = img_to_array(face)
        face = preprocess_input(face)

        # add the face and bounding boxes to their respective lists
        faces.append(face)
        locs.append((startX, startY, endX, endY))
      ```
6. Only make a predictions if at least one face was detected
   ```python
   if len(faces) > 0:
    # for faster inference we'll make batch predictions on **all** faces at the same time rather than one-by-one predictions in the above `for` loop
    faces = np.array(faces, dtype="float32")
    preds = maskNet.predict(faces, batch_size=32)
   ```
7. Return a 2-tuple of the face locations and their corresponding locations
   ```python
   return (locs, preds)
   ```
8. Load our serialized face detector model from disk
   ```python
   prototxtPath = r"face_detector\deploy.prototxt"
   weightsPath = r"face_detector\res10_300x300_ssd_iter_140000.caffemodel"
   faceNet = cv2.dnn.readNet(prototxtPath, weightsPath)
   ```
9. Load our mask detector model from disk
   ```python
   maskNet = load_model("mask_detector.model")
   ```
10. Initialize the video stream
    ```python
    print("[INFO] starting video stream...")
    vs = VideoStream(src=0).start()
    ```
11. Grab the frame from the threaded video stream and resize it to have a maximum width of 900 pixels
    ```python
    while True:
      frame = vs.read()
      frame = imutils.resize(frame, width=900)

      # detect faces in the frame and determine if they are wearing a face mask or not
      (locs, preds) = detect_and_predict_mask(frame, faceNet, maskNet)
    ```                                      
12. Loop over the detected face locations and their corresponding locations
    ```python
    for (box, pred) in zip(locs, preds):
        # unpack the bounding box and predictions
        (startX, startY, endX, endY) = box
        (mask, withoutMask) = pred

        # determine the class label and color we'll use to draw
        # the bounding box and text
        label = "Mask" if mask > withoutMask else "No Mask"
        color = (0, 255, 0) if label == "Mask" else (0, 0, 255)

        # include the probability in the label
        label = "{}: {:.2f}%".format(label, max(mask, withoutMask) * 100)

        # display the label and bounding box rectangle on the output frame
        cv2.putText(frame, label, (startX, startY - 10),
                    cv2.FONT_HERSHEY_SIMPLEX, 0.45, color, 2)
        cv2.rectangle(frame, (startX, startY), (endX, endY), color, 2)
    ```
13. Show the output frame
    ```python
    cv2.imshow("Frame", frame)
    key = cv2.waitKey(1) & 0xFF
    ```
14. If the `q` key was pressed, break from the loop
    ```python
    if key == ord("q"):
    break
    ```
15. Destroy all windows
    ```python
    cv2.destroyAllWindows()
    vs.stop()
    ```
