# FacialDetectionESP32


 ### Facial Detection based on the ROI Prokyber AI-On-The-Edge-Cam prototype (ESP32-S3 N16R8 + OV2640)
 ### Edge Impulse FOMO object detection.
 ### Board settings: 
    PSRAM = OPI, 
    Partition = 16M Flash (3MB APP/9.9MB FATFS),
    CPU = 240MHz WiFi, Board = ESP32S3 Dev Module
 
 # RECOMMENDED EDIT (MEANT FOR ANY NEW SKETCH IN ARDUINO IDE):
    In libraries/<project>_inferencing/src/edge-impulse-sdk/classifier/
    ei_classifier_config.h, at the top, next to the other defines, the following needs to be added:
         #define EI_CLASSIFIER_TFLITE_ENABLE_ESP_NN 0
 * Add this library export from Studio - exporting overwrites

 # SNAPSHOT_MODE (bottom) switches sketch mode:
    1 = one JPEG, print as base64, stop. To check what the camera truly sees
         Paste the line in a browser window.
    0 = normal live inference.
