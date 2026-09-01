#include <ObjectDetection3_inferencing.h>
#include "esp_camera.h"
#include "img_converters.h"
#include "edge-impulse-sdk/dsp/image/image.hpp"
#include "base64.h"

// 1 , 0
#define SNAPSHOT_MODE 0

// Inference are nevoie de mai mult decat default-ul de 8KB per loop stack.
SET_LOOP_TASK_STACK_SIZE(32 * 1024);

// ---------- Pinout ----------
#define PER_EN_PIN      46
#define XCLK_GPIO_NUM   15
#define PCLK_GPIO_NUM   13
#define VSYNC_GPIO_NUM  6
#define HREF_GPIO_NUM   7
#define SIOD_GPIO_NUM   4
#define SIOC_GPIO_NUM   5
#define Y2_GPIO_NUM     11
#define Y3_GPIO_NUM     9
#define Y4_GPIO_NUM     8
#define Y5_GPIO_NUM     10
#define Y6_GPIO_NUM     47
#define Y7_GPIO_NUM     18
#define Y8_GPIO_NUM     17
#define Y9_GPIO_NUM     16

#define CAM_W 320
#define CAM_H 240

// 1, 0. Diagnostic
#define DEBUG_DIAGNOSTICS 0

static uint8_t *snapshot_buf = nullptr;   // RGB888 working buffer, in PSRAM

// ---------------------------------------------------------------------------
// Edge Impulse pulls pixels through this callback: one float per pixel,
// packed 0x00RRGGBB. Grayscale models convert inside the DSP block.
// ---------------------------------------------------------------------------
static int ei_get_data(size_t offset, size_t length, float *out_ptr) {
  size_t px = offset * 3;
  for (size_t i = 0; i < length; i++) {
    out_ptr[i] = (snapshot_buf[px] << 16) + (snapshot_buf[px + 1] << 8) + snapshot_buf[px + 2];
    px += 3;
  }
  return 0;
}

// ---------------------------------------------------------------------------
// Grab a frame, RGB565 -> RGB888, then centre-crop and resize to model input.
// crop_and_interpolate_rgb888 is documented safe in place.
// ---------------------------------------------------------------------------
static bool capture_frame(uint32_t out_w, uint32_t out_h) {
  camera_fb_t *fb = esp_camera_fb_get();
  if (!fb) {
    Serial.println("fb_get failed");
    return false;
  }

  bool ok = fmt2rgb888(fb->buf, fb->len, PIXFORMAT_RGB565, snapshot_buf);
  esp_camera_fb_return(fb);
  if (!ok) {
    Serial.println("RGB888 conversion failed");
    return false;
  }

  if (out_w != CAM_W || out_h != CAM_H) {
    ei::image::processing::crop_and_interpolate_rgb888(
        snapshot_buf, CAM_W, CAM_H,
        snapshot_buf, out_w, out_h);
  }
  return true;
}

static bool camera_start() {
  pinMode(PER_EN_PIN, OUTPUT);
  digitalWrite(PER_EN_PIN, HIGH);      // enable 3.3V_PER rail
  delay(500);

  camera_config_t config = {};
  config.ledc_channel = LEDC_CHANNEL_0;
  config.ledc_timer   = LEDC_TIMER_0;
  config.pin_d0 = Y2_GPIO_NUM;
  config.pin_d1 = Y3_GPIO_NUM;
  config.pin_d2 = Y4_GPIO_NUM;
  config.pin_d3 = Y5_GPIO_NUM;
  config.pin_d4 = Y6_GPIO_NUM;
  config.pin_d5 = Y7_GPIO_NUM;
  config.pin_d6 = Y8_GPIO_NUM;
  config.pin_d7 = Y9_GPIO_NUM;
  config.pin_xclk     = XCLK_GPIO_NUM;
  config.pin_pclk     = PCLK_GPIO_NUM;
  config.pin_vsync    = VSYNC_GPIO_NUM;
  config.pin_href     = HREF_GPIO_NUM;
  config.pin_sccb_sda = SIOD_GPIO_NUM;
  config.pin_sccb_scl = SIOC_GPIO_NUM;
  config.pin_pwdn     = -1;
  config.pin_reset    = -1;
  config.xclk_freq_hz = 20000000;   // drop to 10000000 if VSYNC-OVF persists
  config.frame_size   = FRAMESIZE_QVGA;
  config.fb_count     = 2;
  config.fb_location  = CAMERA_FB_IN_PSRAM;
  config.grab_mode    = CAMERA_GRAB_LATEST;

#if SNAPSHOT_MODE
  config.pixel_format = PIXFORMAT_JPEG;
  config.jpeg_quality = 12;          // lower number = better quality
#else
  config.pixel_format = PIXFORMAT_RGB565;
#endif

  esp_err_t err = esp_camera_init(&config);
  if (err != ESP_OK) {
    Serial.printf("Camera init failed 0x%x\n", err);
    return false;
  }

  sensor_t *s = esp_camera_sensor_get();
  if (s) {
    s->set_vflip(s, 0);         //vflip si hmirror 1 => flip la imagine
    s->set_hmirror(s, 0);
    s->set_whitebal(s, 1);      // auto white balance
    s->set_awb_gain(s, 1);
    s->set_wb_mode(s, 3);       // 0 auto, 1 sunny, 2 cloudy, 3 office, 4 home
    s->set_exposure_ctrl(s, 1); // auto exposure
    s->set_gain_ctrl(s, 1);     // auto gain
    s->set_ae_level(s, 2);      // -2..2, biases auto-exposure brighter
    s->set_brightness(s, 2);    // -2..2
    s->set_contrast(s, 2);      // -2..2
  }

  // OV2640 auto-exposure are nevoie de cateva poze pentru a se obisnui, primele ies ca fiind negre.
  for (int i = 0; i < 5; i++) {
    camera_fb_t *f = esp_camera_fb_get();
    if (f) esp_camera_fb_return(f);
    delay(100);
  }
  return true;
}

void setup() {
  Serial.begin(115200);
  delay(1000);

  if (!camera_start()) {
    Serial.println("Halting.");
    while (true) delay(1000);
  }

  neopixelWrite(12, 150, 150, 150);  // dim white on LED 1 (Blit)
  delay(1000);                      // let AEC settle

#if SNAPSHOT_MODE
  // One JPEG, printed as a data: URL. Copy everything between the markers
  // (one single line, no breaks) into a browser address bar.
  camera_fb_t *fb = esp_camera_fb_get();
  if (fb) {
    Serial.printf("\nJPEG size: %u bytes (expect roughly 5-15 KB)\n", fb->len);
    Serial.println("---BEGIN---");
    Serial.println("data:image/jpeg;base64," + base64::encode(fb->buf, fb->len));
    Serial.println("---END---");
    esp_camera_fb_return(fb);
  } else {
    Serial.println("fb_get failed");
  }
  Serial.println("\nSnapshot done. Set SNAPSHOT_MODE 0 to run inference.");

#else
  snapshot_buf = (uint8_t *)ps_malloc(CAM_W * CAM_H * 3);
  if (!snapshot_buf) {
    Serial.println("PSRAM alloc failed. Is PSRAM set to OPI?");
    while (true) delay(1000);
  }

  Serial.printf("Model input: %dx%d\n",
                EI_CLASSIFIER_INPUT_WIDTH, EI_CLASSIFIER_INPUT_HEIGHT);
  Serial.println("Ready.\n");
#endif
}

void loop() {
#if SNAPSHOT_MODE
  delay(1000);          // reset la board pentru a primi inca un frame
  return;
#else

  if (!capture_frame(EI_CLASSIFIER_INPUT_WIDTH, EI_CLASSIFIER_INPUT_HEIGHT)) {
    delay(500);
    return;
  }

#if DEBUG_DIAGNOSTICS
  {
    // Poza "indoors" se situeaza intre 60-180px. Aproape de 0 sau 255 => senzor stricat nu o problema cu modelul
    const size_t n = EI_CLASSIFIER_INPUT_WIDTH * EI_CLASSIFIER_INPUT_HEIGHT * 3;
    uint32_t sum = 0;
    for (size_t i = 0; i < n; i++) sum += snapshot_buf[i];
    Serial.printf("mean px: %u | stack: %u B | ",
                  (unsigned)(sum / n), uxTaskGetStackHighWaterMark(NULL));
  }
#endif

  ei::signal_t signal;
  signal.total_length = EI_CLASSIFIER_INPUT_WIDTH * EI_CLASSIFIER_INPUT_HEIGHT;
  signal.get_data     = &ei_get_data;

  ei_impulse_result_t result = { 0 };
  EI_IMPULSE_ERROR err = run_classifier(&signal, &result, false);
  if (err != EI_IMPULSE_OK) {
    Serial.printf("run_classifier failed (%d)\n", err);
    delay(500);
    return;
  }

  Serial.printf("[dsp %d ms | inference %d ms] ",
                result.timing.dsp, result.timing.classification);

#if EI_CLASSIFIER_OBJECT_DETECTION == 1
  // FOMO gives centroids, not tight boxes. x,y are in model-input pixels.
  // Note: this array only ever holds detections that already passed the
  // model's confidence threshold, so an empty array tells you nothing about
  // how close it came.
  if (result.bounding_boxes_count == 0) {
    Serial.print("no objects");
  }
  for (uint32_t i = 0; i < result.bounding_boxes_count; i++) {
    ei_impulse_result_bounding_box_t bb = result.bounding_boxes[i];
    if (bb.value == 0) continue;
    Serial.printf("%s %.2f @ (%u,%u) ", bb.label, bb.value, bb.x, bb.y);
  }
  Serial.println();
#else
  for (uint16_t i = 0; i < EI_CLASSIFIER_LABEL_COUNT; i++) {
    Serial.printf("%s: %.3f  ", ei_classifier_inferencing_categories[i],
                  result.classification[i].value);
  }
  Serial.println();
#endif

  delay(100);
#endif  // SNAPSHOT_MODE
}
