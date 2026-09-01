# FacialDetectionESP32


 * Detectie fata pe prototipul ROI Prokyber AI-On-The-Edge-Cam (ESP32-S3 N16R8 + OV2640).
 * Edge Impulse FOMO object detection.
 *
 * Setari placa: PSRAM = OPI, Partition = 16M Flash (3MB APP/9.9MB FATFS),
 *                 CPU = 240MHz WiFi, Board = ESP32S3 Dev Module
 *
 * EDIT RECOMANDAT (PENTRU FIECARE SKETCH NOU):
 *   In libraries/<project>_inferencing/src/edge-impulse-sdk/classifier/
 *   ei_classifier_config.h, sus de tot langa celelalte define-uri trebuie adaugat:
 *       #define EI_CLASSIFIER_TFLITE_ENABLE_ESP_NN 0
 *   Adauga aceast library export din Studio - exporting overwrites
 *
 * SNAPSHOT_MODE (jos) schimba modul sketch-ului:
 *   1 = un JPEG, print ca base64, stop. Pentru a vedea ce vede camera cu adevarat
 *       Paste la linia pozei in browser.
 *   0 = normal live inference.
 * 

