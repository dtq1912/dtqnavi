#include <ChronosESP32.h>
#include <Wire.h>
#include <Adafruit_GFX.h>
#include <Adafruit_SH110X.h>
#include <FontMaker.h> // FontMaker và font DTQ đã tạo

// Định nghĩa kích thước màn hình
#define SCREEN_WIDTH 128
#define SCREEN_HEIGHT 64

// Tạo đối tượng SH1106
Adafruit_SH1106G display(SCREEN_WIDTH, SCREEN_HEIGHT, &Wire, -1);

ChronosESP32 watch("Chronos Nav"); // Tên Bluetooth

bool showNavigation = false;
String navData[4]; // Dữ liệu điều hướng

// Hàm để vẽ pixel cho FontMaker
void setpx(int16_t x, int16_t y, uint16_t color) {
    display.drawPixel(x, y, color); // Hàm vẽ pixel của Adafruit_GFX
}

// Khởi tạo FontMaker
MakeFont myfont(&setpx);

// Hàm hiển thị tất cả thông tin điều hướng
void displayNavigationData() {
    display.clearDisplay(); // Xóa màn hình

    // Sử dụng font DTQ để hiển thị
    myfont.set_font(DTQ4);

    // Hiển thị từng dòng thông tin
    // Dòng 1: Hướng rẽ
    myfont.print(0, 0, navData[0], SH110X_WHITE, SH110X_BLACK); // Ví dụ: "Rẽ Trái"

    // Dòng 2: Tiêu đề điều hướng
    myfont.print(0, 15, navData[1], SH110X_WHITE, SH110X_BLACK); // Ví dụ: "Ng. 24 Đ. Cao Lỗ"

    // Dòng 3: TG (Thời gian)
    myfont.print(0, 30, ("TG: " + navData[2]).c_str(), SH110X_WHITE, SH110X_BLACK); // Ví dụ: "TG: 7 phút"

    // Dòng 4: KC (Khoảng cách)
    myfont.print(0, 45, ("KC: " + navData[3]).c_str(), SH110X_WHITE, SH110X_BLACK); // Ví dụ: "KC: 3,5 km"

    // Cập nhật màn hình
    display.display();
}

void resetDisplay() {
    display.clearDisplay();

    // Sử dụng font DTQ để hiển thị "Chưa kết nối"
    myfont.set_font(DTQ);
    myfont.print(0, 0, "Chưa kết nối", SH110X_WHITE, SH110X_BLACK);

    // Cập nhật màn hình
    display.display();

    // Reset dữ liệu điều hướng
    for (int i = 0; i < 4; i++) {
        navData[i] = "";
    }
    showNavigation = false;
}

// Callback khi nhận dữ liệu điều hướng mới
void configCallback(Config config, uint32_t a, uint32_t b) {
    switch (config) {
    case CF_NAV_DATA:
        Serial.print("Navigation state: ");
        Serial.println(a ? "Active" : "Inactive");

        if (a) {
            Navigation nav = watch.getNavigation();
            Serial.println(nav.directions);
            Serial.println(nav.duration);
            Serial.println(nav.distance);
            Serial.println(nav.title);

            // Phân tích hướng rẽ từ nav.directions
            String directions = nav.directions; // Lấy nội dung điều hướng
            String turnDirection = "Không rõ hướng"; // Mặc định nếu không xác định được

            if (directions.indexOf("left") >= 0) {
                turnDirection = "Rẽ Trái";
            } else if (directions.indexOf("right") >= 0) {
                turnDirection = "Rẽ Phải";
            } else if (directions.indexOf("straight") >= 0) {
                turnDirection = "Tiến Thẳng";
            }

            // Lưu dữ liệu điều hướng
            navData[0] = turnDirection;        // Hướng rẽ (Trái/Phải/Thẳng)
            navData[1] = directions;          // Ví dụ: "Ng. 24 Đ. Cao Lỗ"
            navData[2] = nav.duration;        // TG (Thời lượng)
            navData[3] = nav.distance;        // KC (Khoảng cách)
            showNavigation = true;

            // Hiển thị toàn bộ dữ liệu
            displayNavigationData();
        } else {
            resetDisplay();
        }
        break;

    case CF_NAV_ICON:
        Serial.print("Navigation Icon data, position: ");
        Serial.println(a);
        Serial.print("Icon CRC: ");
        Serial.printf("0x%04X\n", b);
        break;
    }
}

void setup() {
    Serial.begin(115200);

    // Khởi tạo OLED
    if (!display.begin(0x3C)) { // Địa chỉ SH1106
        Serial.println("SH1106 allocation failed");
        for (;;);
    }
    display.clearDisplay();

    // Đặt font ban đầu
    myfont.set_font(DTQ);
    myfont.print(0, 0, "Initializing...", SH110X_WHITE, SH110X_BLACK);
    display.display();

    // Đặt callback
    watch.setConfigurationCallback(configCallback);

    watch.begin(); // Khởi tạo BLE
    Serial.println(watch.getAddress()); // Địa chỉ MAC

    watch.setBattery(80); // Cài đặt mức pin
    resetDisplay();       // Đặt màn hình ban đầu
}

void loop() {
    watch.loop(); // Xử lý BLE

    // Hiển thị dữ liệu điều hướng nếu có
    if (showNavigation) {
        displayNavigationData();
    }
}
