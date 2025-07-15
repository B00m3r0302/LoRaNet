# LoRa Phone Development Plan

## Project Overview
Build a Raspberry Pi 3B-based LoRa communication device that functions like a "phone" with automatic fallback between P2P LoRa and LoRaWAN modes.

## Hardware Requirements
- [x] Raspberry Pi 3B+
- [ ] Pi Supply LoRa Gateway HAT
- [ ] 3.5" TFT Display or OLED (320x240 recommended)
- [ ] USB Keyboard (compact wireless preferred)
- [ ] MicroSD Card (32GB+)
- [ ] LoRa Antenna
- [ ] Power bank/battery solution
- [ ] Optional: GPS module for location services

## Development Environment Setup

### Phase 0: Environment Preparation
- [ ] **0.1** Flash Raspberry Pi OS to SD card
- [ ] **0.2** Enable SSH, SPI, I2C in raspi-config
- [ ] **0.3** Install development tools
  ```bash
  sudo apt update && sudo apt upgrade -y
  sudo apt install build-essential cmake git vim
  sudo apt install libncurses5-dev libsdl2-dev libsdl2-ttf-dev
  sudo apt install wiringpi libwiringpi-dev
  ```
- [ ] **0.4** Set up Git repository
  ```bash
  git init
  git remote add origin <your-repo-url>
  git config user.name "Your Name"
  git config user.email "your.email@example.com"
  ```
- [ ] **0.5** Create project directory structure
- [ ] **0.6** Install RadioHead library

#### Linux Installation:
```bash
# Clone the RadioHead repository
cd ~
git clone https://github.com/PaulStoffregen/RadioHead.git

# Option A: System-wide install (headers only)
sudo cp -r RadioHead/*.h /usr/local/include/
sudo cp -r RadioHead/*.cpp /usr/local/include/
# RadioHead is header-only but includes .cpp files that need to be accessible

# Option B: Per-project include (recommended)
# In your project's CMakeLists.txt, add:
#   include_directories("${HOME}/RadioHead")
# Or copy RadioHead folder to your project:
#   cp -r ~/RadioHead ./libs/
```

#### Windows Installation:
```cmd
# Using Command Prompt or PowerShell
cd %USERPROFILE%
git clone https://github.com/PaulStoffregen/RadioHead.git

# Option A: Copy to project directory (recommended)
# Copy the RadioHead folder to your project's libs directory
# xcopy RadioHead "C:\path\to\your\project\libs\RadioHead" /E /I

# Option B: System-wide with MinGW/MSYS2
# Copy RadioHead contents to your toolchain's include directory
# copy RadioHead\*.h C:\msys64\mingw64\include\
# copy RadioHead\*.cpp C:\msys64\mingw64\include\

# In CMakeLists.txt for Windows:
#   include_directories("${CMAKE_SOURCE_DIR}/libs/RadioHead")
#   or
#   include_directories("$ENV{USERPROFILE}/RadioHead")
```
## Project Structure
```
lora_phone/
├── README.md
├── CMakeLists.txt
├── Makefile
├── src/
│   ├── main.cpp
│   ├── lora/
│   │   ├── radio_manager.cpp
│   │   ├── p2p_protocol.cpp
│   │   ├── lorawan_protocol.cpp
│   │   └── mesh_routing.cpp
│   ├── ui/
│   │   ├── tui_manager.cpp
│   │   ├── gui_manager.cpp
│   │   ├── display_driver.cpp
│   │   └── input_handler.cpp
│   ├── messaging/
│   │   ├── message_queue.cpp
│   │   ├── contact_manager.cpp
│   │   ├── encryption.cpp
│   │   └── message_parser.cpp
│   ├── network/
│   │   ├── network_manager.cpp
│   │   ├── discovery_service.cpp
│   │   └── topology_manager.cpp
│   └── utils/
│       ├── config.cpp
│       ├── logger.cpp
│       ├── storage.cpp
│       └── time_utils.cpp
├── include/
│   ├── lora/
│   │   ├── radio_manager.h
│   │   ├── protocols.h
│   │   └── mesh_routing.h
│   ├── ui/
│   │   ├── ui_manager.h
│   │   ├── display_driver.h
│   │   └── input_handler.h
│   ├── messaging/
│   │   ├── message_queue.h
│   │   ├── contact_manager.h
│   │   ├── encryption.h
│   │   └── message_types.h
│   ├── network/
│   │   ├── network_manager.h
│   │   ├── discovery_service.h
│   │   └── topology_manager.h
│   └── utils/
│       ├── config.h
│       ├── logger.h
│       ├── storage.h
│       └── common.h
├── libs/
│   ├── RadioHead/          # Copied from git clone
│   ├── json/               # nlohmann/json
│   └── sqlite/             # SQLite amalgamation
├── config/
│   ├── default_config.json
│   ├── contacts.json
│   └── network_config.json
├── docs/
│   ├── API.md
│   ├── PROTOCOL.md
│   ├── USAGE.md
│   └── BUILD.md
├── tests/
│   ├── unit/
│   │   ├── test_radio.cpp
│   │   ├── test_messaging.cpp
│   │   ├── test_ui.cpp
│   │   └── test_encryption.cpp
│   ├── integration/
│   │   ├── test_e2e.cpp
│   │   └── test_network.cpp
│   └── mocks/
│       ├── mock_radio.h
│       └── mock_storage.h
├── scripts/
│   ├── build.sh
│   ├── install.sh
│   └── setup_env.sh
└── data/
    ├── messages.db         # SQLite database
    └── logs/
        └── app.log
```

## Detailed File Specifications

### Main Application Files

#### **src/main.cpp**
Entry point and main application loop.

**Key Components:**
- Application initialization
- Main event loop
- Signal handling
- Cleanup procedures

**Required Libraries:**
```cpp
#include <iostream>
#include <memory>
#include <signal.h>
#include <unistd.h>
#include "utils/logger.h"
#include "utils/config.h"
#include "lora/radio_manager.h"
#include "ui/ui_manager.h"
#include "messaging/message_queue.h"
#include "network/network_manager.h"
```

**Key Functions:**
```cpp
int main(int argc, char* argv[]);
void signalHandler(int signal);
bool initializeApplication();
void runMainLoop();
void cleanupApplication();
```

#### **CMakeLists.txt**
Build configuration with all dependencies.

**Contents:**
```cmake
cmake_minimum_required(VERSION 3.16)
project(lora_phone VERSION 1.0.0)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Find packages
find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)

# Check for ncurses
pkg_check_modules(NCURSES REQUIRED ncurses)

# Include directories
include_directories(
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/libs/RadioHead
    ${CMAKE_SOURCE_DIR}/libs/json/include
    ${CMAKE_SOURCE_DIR}/libs/sqlite
)

# Source files
set(SOURCES
    src/main.cpp
    src/lora/radio_manager.cpp
    src/lora/p2p_protocol.cpp
    src/lora/lorawan_protocol.cpp
    src/lora/mesh_routing.cpp
    src/ui/tui_manager.cpp
    src/ui/display_driver.cpp
    src/ui/input_handler.cpp
    src/messaging/message_queue.cpp
    src/messaging/contact_manager.cpp
    src/messaging/encryption.cpp
    src/messaging/message_parser.cpp
    src/network/network_manager.cpp
    src/network/discovery_service.cpp
    src/network/topology_manager.cpp
    src/utils/config.cpp
    src/utils/logger.cpp
    src/utils/storage.cpp
    src/utils/time_utils.cpp
    libs/sqlite/sqlite3.c
)

# RadioHead sources
file(GLOB RADIOHEAD_SOURCES "${CMAKE_SOURCE_DIR}/libs/RadioHead/*.cpp")
list(APPEND SOURCES ${RADIOHEAD_SOURCES})

# Create executable
add_executable(${PROJECT_NAME} ${SOURCES})

# Link libraries
target_link_libraries(${PROJECT_NAME}
    ${NCURSES_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT}
    OpenSSL::SSL
    OpenSSL::Crypto
    wiringPi
    rt
    dl
)

# Compiler flags
target_compile_options(${PROJECT_NAME} PRIVATE
    -Wall -Wextra -Wpedantic
    -O2
    -DRASPBERRY_PI
    -D__linux__
)

# Install target
install(TARGETS ${PROJECT_NAME} DESTINATION bin)
install(DIRECTORY config/ DESTINATION etc/${PROJECT_NAME})
```

---

## Phase 1: Core Radio Communication (Weeks 1-2)

### Milestone 1.1: Basic LoRa P2P Communication

#### **1.1.1** Create `RadioManager` class
**Files to create:**
- `include/lora/radio_manager.h`
- `src/lora/radio_manager.cpp`

**Implementation requirements:**
```cpp
// src/lora/radio_manager.cpp
#include "lora/radio_manager.h"
#include <wiringPi.h>
#include <wiringPiSPI.h>

RadioManager::RadioManager() : currentMode(RadioMode::P2P), currentState(RadioState::IDLE) {
    // Initialize wiringPi
    if (wiringPiSetup() == -1) {
        LOG_ERROR("Failed to initialize wiringPi");
        return;
    }
    
    // Setup SPI
    if (wiringPiSPISetup(0, 1000000) == -1) {
        LOG_ERROR("Failed to initialize SPI");
        return;
    }
    
    // Create radio instance
    radio = std::make_unique<RH_RF95>(CS_PIN, INT_PIN);
    reliableRadio = std::make_unique<RHReliableDatagram>(*radio, 1); // Node ID 1
}

bool RadioManager::initialize(const RadioConfig& cfg) {
    config = cfg;
    
    // Reset radio
    pinMode(RST_PIN, OUTPUT);
    digitalWrite(RST_PIN, LOW);
    delay(10);
    digitalWrite(RST_PIN, HIGH);
    delay(10);
    
    // Initialize radio
    if (!radio->init()) {
        LOG_ERROR("Radio initialization failed");
        return false;
    }
    
    // Configure radio parameters
    if (!configureRadio()) {
        LOG_ERROR("Radio configuration failed");
        return false;
    }
    
    // Start receive thread
    running = true;
    rxThread = std::thread(&RadioManager::rxThreadFunction, this);
    
    LOG_INFO("Radio initialized successfully");
    return true;
}

bool RadioManager::configureRadio() {
    // Set frequency
    if (!radio->setFrequency(config.frequency)) {
        LOG_ERROR("Failed to set frequency");
        return false;
    }
    
    // Set modem configuration
    if (!radio->setModemConfig(RH_RF95::Bw125Cr45Sf128)) {
        LOG_ERROR("Failed to set modem config");
        return false;
    }
    
    // Set TX power
    radio->setTxPower(config.txPower, false);
    
    // Enable CRC
    radio->setCrcEnable(config.crc);
    
    LOG_INFO("Radio configured: freq=%.1f MHz, SF=%d, BW=%d kHz, power=%d dBm", 
             config.frequency, config.spreadingFactor, config.bandwidth, config.txPower);
    return true;
}
```

**Testing requirements:**
- Unit test for radio initialization
- Test SPI communication
- Verify radio configuration parameters
- Test reset sequence

#### **1.1.2** Create simple message protocol
**Files to create:**
- `include/messaging/message_types.h` (already specified)
- `src/messaging/message_parser.cpp`

**Implementation requirements:**
```cpp
// src/messaging/message_parser.cpp
#include "messaging/message_types.h"
#include <cstring>
#include <chrono>

std::vector<uint8_t> Message::serialize() const {
    std::vector<uint8_t> data;
    data.reserve(sizeof(MessageHeader) + payload.size());
    
    // Serialize header
    const uint8_t* headerBytes = reinterpret_cast<const uint8_t*>(&header);
    data.insert(data.end(), headerBytes, headerBytes + sizeof(MessageHeader));
    
    // Serialize payload
    data.insert(data.end(), payload.begin(), payload.end());
    
    return data;
}

Message Message::deserialize(const uint8_t* data, size_t len) {
    Message msg;
    
    if (len < sizeof(MessageHeader)) {
        LOG_ERROR("Message too short for header");
        return msg;
    }
    
    // Deserialize header
    std::memcpy(&msg.header, data, sizeof(MessageHeader));
    
    // Verify magic number
    if (msg.header.magic != 0x4C52) {
        LOG_ERROR("Invalid magic number");
        return msg;
    }
    
    // Deserialize payload
    if (len > sizeof(MessageHeader)) {
        size_t payloadSize = len - sizeof(MessageHeader);
        msg.payload.resize(payloadSize);
        std::memcpy(msg.payload.data(), data + sizeof(MessageHeader), payloadSize);
    }
    
    // Verify CRC
    if (!msg.isValid()) {
        LOG_ERROR("Message CRC validation failed");
    }
    
    return msg;
}

bool Message::isValid() const {
    uint32_t calculatedCrc = calculateCRC();
    return calculatedCrc == header.crc32;
}

uint32_t Message::calculateCRC() const {
    // Simple CRC32 calculation
    uint32_t crc = 0xFFFFFFFF;
    
    // CRC over header (excluding CRC field)
    const uint8_t* headerBytes = reinterpret_cast<const uint8_t*>(&header);
    for (size_t i = 0; i < sizeof(MessageHeader) - sizeof(uint32_t); i++) {
        crc ^= headerBytes[i];
        for (int j = 0; j < 8; j++) {
            crc = (crc >> 1) ^ (0xEDB88320 & (-(crc & 1)));
        }
    }
    
    // CRC over payload
    for (uint8_t byte : payload) {
        crc ^= byte;
        for (int j = 0; j < 8; j++) {
            crc = (crc >> 1) ^ (0xEDB88320 & (-(crc & 1)));
        }
    }
    
    return ~crc;
}
```

**Message creation helper:**
```cpp
// Helper function to create text messages
Message createTextMessage(const std::string& senderId, const std::string& recipientId, 
                         const std::string& text) {
    Message msg;
    msg.header.magic = 0x4C52;
    msg.header.version = 0x01;
    msg.header.type = MessageType::TEXT;
    msg.header.sequenceNumber = getNextSequenceNumber();
    msg.header.priority = static_cast<uint8_t>(MessagePriority::NORMAL);
    msg.header.timestamp = std::chrono::system_clock::now().time_since_epoch().count();
    
    // Set sender and recipient IDs
    strncpy(reinterpret_cast<char*>(msg.header.senderId), senderId.c_str(), 16);
    strncpy(reinterpret_cast<char*>(msg.header.recipientId), recipientId.c_str(), 16);
    
    // Set payload
    msg.setPayloadFromString(text);
    msg.header.payloadLength = msg.payload.size();
    
    // Calculate CRC
    msg.header.crc32 = msg.calculateCRC();
    
    return msg;
}
```

**Testing requirements:**
- Test message serialization/deserialization
- Verify CRC calculation
- Test different message types
- Validate header fields

#### **1.1.3** Test basic P2P communication
**Files to create:**
- `tests/unit/test_radio_basic.cpp`
- `tests/integration/test_p2p_communication.cpp`

**Implementation requirements:**
```cpp
// tests/integration/test_p2p_communication.cpp
#include <gtest/gtest.h>
#include "lora/radio_manager.h"
#include "messaging/message_types.h"

class P2PCommTest : public ::testing::Test {
protected:
    void SetUp() override {
        radioManager = std::make_unique<RadioManager>();
        ASSERT_TRUE(radioManager->initialize());
    }
    
    void TearDown() override {
        radioManager->shutdown();
    }
    
    std::unique_ptr<RadioManager> radioManager;
};

TEST_F(P2PCommTest, SendHelloWorld) {
    // Create test message
    Message testMsg = createTextMessage("TEST_001", "TEST_002", "Hello World");
    
    // Send message
    bool sent = radioManager->sendMessage(testMsg, 2);
    EXPECT_TRUE(sent);
    
    // Verify statistics
    auto stats = radioManager->getStats();
    EXPECT_EQ(stats.packetsSent, 1);
}

TEST_F(P2PCommTest, ReceiveMessage) {
    // Set up message callback
    bool messageReceived = false;
    Message receivedMsg;
    
    radioManager->setMessageCallback([&](const Message& msg) {
        messageReceived = true;
        receivedMsg = msg;
    });
    
    // Wait for message (in real test, another device would send)
    std::this_thread::sleep_for(std::chrono::seconds(5));
    
    // Verify message received
    if (messageReceived) {
        EXPECT_EQ(receivedMsg.header.type, MessageType::TEXT);
        EXPECT_TRUE(receivedMsg.isValid());
        EXPECT_GT(receivedMsg.rssi, -150);  // Reasonable RSSI value
    }
}

TEST_F(P2PCommTest, RSSIAndSNRValues) {
    // Test RSSI and SNR reporting
    auto stats = radioManager->getStats();
    
    // After receiving a message, check signal quality
    if (stats.packetsReceived > 0) {
        EXPECT_GT(stats.lastRSSI, -150);
        EXPECT_LT(stats.lastRSSI, 0);
        EXPECT_GT(stats.lastSNR, -20.0);
    }
}
```

**Manual testing procedure:**
```bash
# Create test program
# tests/manual/test_range.cpp
int main() {
    RadioManager radio;
    if (!radio.initialize()) {
        std::cerr << "Failed to initialize radio" << std::endl;
        return 1;
    }
    
    // Test 1: Close range (1-2 meters)
    std::cout << "Testing close range communication..." << std::endl;
    Message msg = createTextMessage("TEST_A", "TEST_B", "Close range test");
    radio.sendMessage(msg, 2);
    
    // Test 2: Medium range (10-50 meters)
    std::cout << "Testing medium range communication..." << std::endl;
    msg = createTextMessage("TEST_A", "TEST_B", "Medium range test");
    radio.sendMessage(msg, 2);
    
    // Test 3: Long range (100+ meters)
    std::cout << "Testing long range communication..." << std::endl;
    msg = createTextMessage("TEST_A", "TEST_B", "Long range test");
    radio.sendMessage(msg, 2);
    
    // Print statistics
    auto stats = radio.getStats();
    std::cout << "Sent: " << stats.packetsSent << std::endl;
    std::cout << "Received: " << stats.packetsReceived << std::endl;
    std::cout << "Last RSSI: " << stats.lastRSSI << " dBm" << std::endl;
    std::cout << "Last SNR: " << stats.lastSNR << " dB" << std::endl;
    
    return 0;
}
```

**Testing requirements:**
- Two devices for P2P testing
- Range testing at different distances
- RSSI/SNR measurement validation
- Packet loss rate analysis
- Error handling verification

### Milestone 1.2: Message Queue and Reliability

#### **1.2.1** Implement message queue system
**Files to create:**
- `src/messaging/message_queue.cpp`
- `tests/unit/test_message_queue.cpp`

**Implementation requirements:**
```cpp
// src/messaging/message_queue.cpp
#include "messaging/message_queue.h"
#include "utils/logger.h"

MessageQueue::MessageQueue() : running(false) {}

MessageQueue::~MessageQueue() {
    stop();
}

bool MessageQueue::start() {
    if (running.load()) {
        LOG_WARNING("MessageQueue already running");
        return true;
    }
    
    running = true;
    processingThread = std::thread(&MessageQueue::processQueue, this);
    
    LOG_INFO("MessageQueue started");
    return true;
}

void MessageQueue::stop() {
    if (!running.load()) return;
    
    running = false;
    queueCondition.notify_all();
    
    if (processingThread.joinable()) {
        processingThread.join();
    }
    
    LOG_INFO("MessageQueue stopped");
}

bool MessageQueue::enqueue(const Message& msg) {
    std::lock_guard<std::mutex> lock(outboundMutex);
    
    if (outboundQueue.size() >= maxQueueSize) {
        LOG_ERROR("Outbound queue full, dropping message");
        stats.totalDropped++;
        return false;
    }
    
    outboundQueue.push(msg);
    queueCondition.notify_one();
    
    LOG_DEBUG("Message enqueued: type=%d, priority=%d", 
              static_cast<int>(msg.header.type), msg.header.priority);
    return true;
}

bool MessageQueue::enqueueHighPriority(const Message& msg) {
    Message priorityMsg = msg;
    priorityMsg.header.priority = static_cast<uint8_t>(MessagePriority::HIGH);
    return enqueue(priorityMsg);
}

void MessageQueue::processQueue() {
    while (running.load()) {
        std::unique_lock<std::mutex> lock(outboundMutex);
        
        // Wait for messages or stop signal
        queueCondition.wait(lock, [this] { 
            return !outboundQueue.empty() || !running.load(); 
        });
        
        if (!running.load()) break;
        
        if (!outboundQueue.empty()) {
            Message msg = outboundQueue.top();
            outboundQueue.pop();
            lock.unlock();
            
            // Try to send message
            if (sendCallback && sendCallback(msg)) {
                stats.totalSent++;
                LOG_DEBUG("Message sent successfully: seq=%d", msg.header.sequenceNumber);
                
                // Add to pending ACKs if not broadcast
                if (msg.header.type != MessageType::HEARTBEAT) {
                    std::lock_guard<std::mutex> ackLock(ackMutex);
                    pendingAcks.push_back(msg);
                }
            } else {
                // Retry logic
                if (shouldRetry(msg)) {
                    Message retryMsg = msg;
                    retryMsg.retryCount++;
                    retryMsg.lastAttempt = std::chrono::system_clock::now();
                    
                    std::lock_guard<std::mutex> retryLock(outboundMutex);
                    outboundQueue.push(retryMsg);
                    stats.totalRetries++;
                    
                    LOG_DEBUG("Message retry queued: seq=%d, attempt=%d", 
                              msg.header.sequenceNumber, retryMsg.retryCount);
                } else {
                    stats.totalDropped++;
                    LOG_ERROR("Message dropped after max retries: seq=%d", 
                              msg.header.sequenceNumber);
                }
            }
        }
        
        // Clean up old pending ACKs
        processAcks();
    }
}

bool MessageQueue::shouldRetry(const Message& msg) const {
    if (msg.retryCount >= maxRetries) return false;
    
    auto now = std::chrono::system_clock::now();
    auto timeSinceLastAttempt = now - msg.lastAttempt;
    
    return timeSinceLastAttempt >= retryDelay;
}

void MessageQueue::processAcks() {
    std::lock_guard<std::mutex> lock(ackMutex);
    auto now = std::chrono::system_clock::now();
    
    pendingAcks.erase(
        std::remove_if(pendingAcks.begin(), pendingAcks.end(),
            [this, now](const Message& msg) {
                auto age = now - msg.lastAttempt;
                return age > ackTimeout;
            }),
        pendingAcks.end()
    );
}

void MessageQueue::handleAck(const Message& ackMsg) {
    std::lock_guard<std::mutex> lock(ackMutex);
    
    // Find and remove the original message from pending ACKs
    auto it = std::find_if(pendingAcks.begin(), pendingAcks.end(),
        [&ackMsg](const Message& msg) {
            return msg.header.sequenceNumber == ackMsg.header.sequenceNumber;
        });
    
    if (it != pendingAcks.end()) {
        pendingAcks.erase(it);
        LOG_DEBUG("ACK received for message seq=%d", ackMsg.header.sequenceNumber);
    }
}

Message MessageQueue::createAckMessage(const Message& originalMsg) const {
    Message ackMsg;
    ackMsg.header.magic = 0x4C52;
    ackMsg.header.version = 0x01;
    ackMsg.header.type = MessageType::ACK;
    ackMsg.header.sequenceNumber = originalMsg.header.sequenceNumber;
    ackMsg.header.priority = static_cast<uint8_t>(MessagePriority::HIGH);
    ackMsg.header.timestamp = std::chrono::system_clock::now().time_since_epoch().count();
    
    // Swap sender and recipient for ACK
    std::memcpy(ackMsg.header.senderId, originalMsg.header.recipientId, 16);
    std::memcpy(ackMsg.header.recipientId, originalMsg.header.senderId, 16);
    
    ackMsg.header.payloadLength = 0;
    ackMsg.header.crc32 = ackMsg.calculateCRC();
    
    return ackMsg;
}
```

**Testing requirements:**
- Test priority queue ordering
- Verify retry mechanism
- Test queue capacity limits
- Validate ACK handling
- Test thread safety

#### **1.2.2** Add packet validation
**Files to create:**
- `src/messaging/packet_validator.cpp`
- `include/messaging/packet_validator.h`

**Implementation requirements:**
```cpp
// include/messaging/packet_validator.h
#pragma once
#include "message_types.h"
#include <unordered_set>
#include <chrono>

class PacketValidator {
private:
    std::unordered_set<uint32_t> seenMessages;
    std::chrono::system_clock::time_point lastCleanup;
    static constexpr auto CLEANUP_INTERVAL = std::chrono::minutes(5);
    static constexpr auto MESSAGE_TIMEOUT = std::chrono::minutes(10);
    
public:
    bool validateMessage(const Message& msg);
    bool isDuplicate(const Message& msg);
    bool isValidSequence(const Message& msg);
    void cleanupOldMessages();
    
private:
    uint32_t generateMessageHash(const Message& msg) const;
    bool isTimestampValid(uint32_t timestamp) const;
};

// src/messaging/packet_validator.cpp
#include "messaging/packet_validator.h"
#include "utils/logger.h"
#include <ctime>

bool PacketValidator::validateMessage(const Message& msg) {
    // Check magic number
    if (msg.header.magic != 0x4C52) {
        LOG_ERROR("Invalid magic number: 0x%04X", msg.header.magic);
        return false;
    }
    
    // Check version
    if (msg.header.version != 0x01) {
        LOG_ERROR("Unsupported version: %d", msg.header.version);
        return false;
    }
    
    // Validate CRC
    if (!msg.isValid()) {
        LOG_ERROR("CRC validation failed");
        return false;
    }
    
    // Check payload length
    if (msg.header.payloadLength != msg.payload.size()) {
        LOG_ERROR("Payload length mismatch: header=%d, actual=%zu", 
                  msg.header.payloadLength, msg.payload.size());
        return false;
    }
    
    // Check timestamp (not too old or future)
    if (!isTimestampValid(msg.header.timestamp)) {
        LOG_ERROR("Invalid timestamp: %u", msg.header.timestamp);
        return false;
    }
    
    // Check for duplicates
    if (isDuplicate(msg)) {
        LOG_DEBUG("Duplicate message detected: seq=%d", msg.header.sequenceNumber);
        return false;
    }
    
    // Clean up old messages periodically
    auto now = std::chrono::system_clock::now();
    if (now - lastCleanup > CLEANUP_INTERVAL) {
        cleanupOldMessages();
        lastCleanup = now;
    }
    
    return true;
}

bool PacketValidator::isDuplicate(const Message& msg) {
    uint32_t hash = generateMessageHash(msg);
    
    if (seenMessages.find(hash) != seenMessages.end()) {
        return true;
    }
    
    seenMessages.insert(hash);
    return false;
}

uint32_t PacketValidator::generateMessageHash(const Message& msg) const {
    // Simple hash based on sender, sequence number, and timestamp
    uint32_t hash = 0;
    
    // Hash sender ID
    for (int i = 0; i < 16; i++) {
        hash ^= msg.header.senderId[i] << (i % 4);
    }
    
    // Hash sequence number and timestamp
    hash ^= msg.header.sequenceNumber << 16;
    hash ^= msg.header.timestamp;
    
    return hash;
}

bool PacketValidator::isTimestampValid(uint32_t timestamp) const {
    auto now = std::chrono::system_clock::now();
    auto msgTime = std::chrono::system_clock::time_point(
        std::chrono::seconds(timestamp));
    
    auto diff = now - msgTime;
    
    // Allow messages up to 10 minutes old or 1 minute in future
    return diff <= std::chrono::minutes(10) && diff >= std::chrono::minutes(-1);
}

void PacketValidator::cleanupOldMessages() {
    // In a real implementation, we would need to track message timestamps
    // For now, just clear old entries periodically
    if (seenMessages.size() > 10000) {
        seenMessages.clear();
        LOG_DEBUG("Cleaned up old message cache");
    }
}
```

**Testing requirements:**
- Test CRC validation
- Test duplicate detection
- Test timestamp validation
- Test message hash generation
- Test cleanup mechanism

#### **1.2.3** Implement heartbeat system
**Files to create:**
- `src/network/heartbeat_manager.cpp`
- `include/network/heartbeat_manager.h`

**Implementation requirements:**
```cpp
// include/network/heartbeat_manager.h
#pragma once
#include "messaging/message_types.h"
#include <chrono>
#include <thread>
#include <atomic>
#include <unordered_map>
#include <functional>

struct DeviceInfo {
    std::string deviceId;
    std::chrono::system_clock::time_point lastSeen;
    int16_t lastRSSI;
    float lastSNR;
    uint32_t heartbeatCount;
    bool isOnline;
};

class HeartbeatManager {
private:
    std::unordered_map<std::string, DeviceInfo> knownDevices;
    std::thread heartbeatThread;
    std::atomic<bool> running;
    std::chrono::milliseconds heartbeatInterval;
    std::chrono::milliseconds deviceTimeout;
    std::function<void(const Message&)> sendCallback;
    std::function<void(const std::string&, bool)> statusCallback;
    std::string deviceId;
    
public:
    HeartbeatManager(const std::string& deviceId, 
                    std::chrono::milliseconds interval = std::chrono::seconds(30));
    ~HeartbeatManager();
    
    bool start();
    void stop();
    
    void handleHeartbeat(const Message& msg);
    void setSendCallback(std::function<void(const Message&)> callback);
    void setStatusCallback(std::function<void(const std::string&, bool)> callback);
    
    std::vector<DeviceInfo> getKnownDevices() const;
    bool isDeviceOnline(const std::string& deviceId) const;
    
private:
    void heartbeatLoop();
    void sendHeartbeat();
    void checkDeviceTimeouts();
    Message createHeartbeatMessage() const;
};

// src/network/heartbeat_manager.cpp
#include "network/heartbeat_manager.h"
#include "utils/logger.h"

HeartbeatManager::HeartbeatManager(const std::string& deviceId, 
                                  std::chrono::milliseconds interval)
    : deviceId(deviceId), heartbeatInterval(interval), 
      deviceTimeout(interval * 3), running(false) {}

HeartbeatManager::~HeartbeatManager() {
    stop();
}

bool HeartbeatManager::start() {
    if (running.load()) {
        LOG_WARNING("HeartbeatManager already running");
        return true;
    }
    
    running = true;
    heartbeatThread = std::thread(&HeartbeatManager::heartbeatLoop, this);
    
    LOG_INFO("HeartbeatManager started with interval %lld ms", 
             heartbeatInterval.count());
    return true;
}

void HeartbeatManager::stop() {
    if (!running.load()) return;
    
    running = false;
    if (heartbeatThread.joinable()) {
        heartbeatThread.join();
    }
    
    LOG_INFO("HeartbeatManager stopped");
}

void HeartbeatManager::heartbeatLoop() {
    while (running.load()) {
        sendHeartbeat();
        checkDeviceTimeouts();
        
        std::this_thread::sleep_for(heartbeatInterval);
    }
}

void HeartbeatManager::sendHeartbeat() {
    if (!sendCallback) return;
    
    Message heartbeat = createHeartbeatMessage();
    sendCallback(heartbeat);
    
    LOG_DEBUG("Heartbeat sent");
}

Message HeartbeatManager::createHeartbeatMessage() const {
    Message msg;
    msg.header.magic = 0x4C52;
    msg.header.version = 0x01;
    msg.header.type = MessageType::HEARTBEAT;
    msg.header.sequenceNumber = 0; // Heartbeats don't need sequence numbers
    msg.header.priority = static_cast<uint8_t>(MessagePriority::LOW);
    msg.header.timestamp = std::chrono::system_clock::now().time_since_epoch().count();
    
    // Set sender ID
    strncpy(reinterpret_cast<char*>(msg.header.senderId), deviceId.c_str(), 16);
    
    // Broadcast heartbeat
    memset(msg.header.recipientId, 0xFF, 16);
    
    // Payload contains device info
    std::string payload = "HEARTBEAT:" + deviceId;
    msg.setPayloadFromString(payload);
    msg.header.payloadLength = msg.payload.size();
    msg.header.crc32 = msg.calculateCRC();
    
    return msg;
}

void HeartbeatManager::handleHeartbeat(const Message& msg) {
    if (msg.header.type != MessageType::HEARTBEAT) return;
    
    std::string senderId(reinterpret_cast<const char*>(msg.header.senderId), 16);
    senderId = senderId.substr(0, senderId.find('\0')); // Remove null padding
    
    auto now = std::chrono::system_clock::now();
    auto it = knownDevices.find(senderId);
    
    if (it != knownDevices.end()) {
        // Update existing device
        it->second.lastSeen = now;
        it->second.lastRSSI = msg.rssi;
        it->second.lastSNR = msg.snr;
        it->second.heartbeatCount++;
        
        if (!it->second.isOnline) {
            it->second.isOnline = true;
            if (statusCallback) {
                statusCallback(senderId, true);
            }
            LOG_INFO("Device %s came online", senderId.c_str());
        }
    } else {
        // New device discovered
        DeviceInfo info;
        info.deviceId = senderId;
        info.lastSeen = now;
        info.lastRSSI = msg.rssi;
        info.lastSNR = msg.snr;
        info.heartbeatCount = 1;
        info.isOnline = true;
        
        knownDevices[senderId] = info;
        
        if (statusCallback) {
            statusCallback(senderId, true);
        }
        LOG_INFO("New device discovered: %s", senderId.c_str());
    }
}

void HeartbeatManager::checkDeviceTimeouts() {
    auto now = std::chrono::system_clock::now();
    
    for (auto& [id, info] : knownDevices) {
        if (info.isOnline && (now - info.lastSeen) > deviceTimeout) {
            info.isOnline = false;
            if (statusCallback) {
                statusCallback(id, false);
            }
            LOG_INFO("Device %s went offline", id.c_str());
        }
    }
}

std::vector<DeviceInfo> HeartbeatManager::getKnownDevices() const {
    std::vector<DeviceInfo> devices;
    for (const auto& [id, info] : knownDevices) {
        devices.push_back(info);
    }
    return devices;
}

bool HeartbeatManager::isDeviceOnline(const std::string& deviceId) const {
    auto it = knownDevices.find(deviceId);
    return it != knownDevices.end() && it->second.isOnline;
}
```

**Testing requirements:**
- Test heartbeat transmission
- Test device discovery
- Test timeout detection
- Test network topology mapping
- Test link quality monitoring

### Milestone 1.3: LoRaWAN Integration Setup
- [ ] **1.3.1** Research LoRaWAN libraries
  - Evaluate LMIC vs RadioHead vs custom implementation
  - Set up The Things Network (TTN) account
  - Configure gateway access
- [ ] **1.3.2** Implement basic LoRaWAN stack
  - Device registration and OTAA join
  - Basic uplink/downlink functionality
  - Duty cycle compliance
- [ ] **1.3.3** Create mode switching logic
  - Detect P2P network availability
  - Automatic fallback to LoRaWAN
  - Seamless mode transitions

**Phase 1 Deliverables:**
- Working P2P LoRa communication
- Basic LoRaWAN connectivity
- Automatic mode switching
- Command-line interface for testing

---

## Phase 2: User Interface Development (Weeks 3-4)

### Milestone 2.1: Terminal User Interface (TUI)
- [ ] **2.1.1** Choose TUI framework
  - Evaluate ncurses, FTXUI, or ImTUI
  - Set up development environment
- [ ] **2.1.2** Create main interface layout
  - Message display area (scrollable)
  - Input field for typing
  - Status bar (mode, signal strength, battery)
  - Menu system for settings
- [ ] **2.1.3** Implement core TUI features
  - Message composition and sending
  - Conversation history
  - Contact selection
  - Settings menu

### Milestone 2.2: Graphical User Interface (GUI) - Optional
- [ ] **2.2.1** Choose GUI framework
  - SDL2 + Dear ImGui (lightweight)
  - Qt (feature-rich but heavier)
  - GTK+ (native Linux feel)
- [ ] **2.2.2** Create touch-friendly interface
  - Large buttons for text input
  - Swipe gestures for navigation
  - Virtual keyboard integration
- [ ] **2.2.3** Implement core GUI features
  - Message bubbles (chat-like interface)
  - Contact list with avatars
  - Settings screens
  - Status indicators

### Milestone 2.3: Display Driver Integration
- [ ] **2.3.1** Set up display hardware
  - Configure SPI/I2C for chosen display
  - Test basic framebuffer operations
  - Implement display rotation/brightness
- [ ] **2.3.2** Integrate with UI framework
  - Render TUI/GUI to display
  - Handle input events (keyboard, touch)
  - Optimize for display resolution
- [ ] **2.3.3** Power management
  - Display sleep/wake functionality
  - Backlight control
  - Low power modes

**Phase 2 Deliverables:**
- Functional TUI with message display
- Optional GUI for touch interaction
- Display driver integration
- User-friendly interface for messaging

---

## Phase 3: Messaging System (Weeks 5-6)

### Milestone 3.1: Message Management
- [ ] **3.1.1** Create message storage system
  - SQLite database for message history
  - Efficient indexing for search
  - Message status tracking (sent, delivered, read)
- [ ] **3.1.2** Implement conversation threading
  - Group messages by conversation
  - Conversation metadata (participants, timestamps)
  - Message sorting and pagination
- [ ] **3.1.3** Add message features
  - Message templates/quick replies
  - Message drafts
  - Message search functionality

### Milestone 3.2: Contact Management
- [ ] **3.2.1** Create contact database
  - Store contact information (ID, name, last seen)
  - Contact groups/categories
  - Favorite contacts
- [ ] **3.2.2** Implement contact discovery
  - Automatic contact detection from messages
  - Manual contact addition
  - Contact synchronization between devices
- [ ] **3.2.3** Add contact features
  - Contact aliases/nicknames
  - Contact blocking functionality
  - Contact status (online, offline, away)

### Milestone 3.3: Encryption and Security
- [ ] **3.3.1** Implement message encryption
  - Choose encryption algorithm (AES-256, ChaCha20)
  - Key exchange mechanism
  - End-to-end encryption for P2P mode
- [ ] **3.3.2** Add authentication
  - Device identity verification
  - Message signing and verification
  - Protection against replay attacks
- [ ] **3.3.3** Security features
  - Secure key storage
  - Perfect forward secrecy
  - Optional: steganography for covert communication

**Phase 3 Deliverables:**
- Persistent message storage
- Contact management system
- End-to-end encryption
- Secure communication protocols

---

## Phase 4: Advanced Features (Weeks 7-8)

### Milestone 4.1: Network Features
- [ ] **4.1.1** Implement mesh networking
  - Multi-hop message routing
  - Dynamic route discovery
  - Load balancing across paths
- [ ] **4.1.2** Add broadcast/multicast
  - Group messaging functionality
  - Emergency broadcast mode
  - Selective message flooding
- [ ] **4.1.3** Network monitoring
  - Network topology visualization
  - Link quality metrics
  - Performance statistics

### Milestone 4.2: Location Services
- [ ] **4.2.1** GPS integration
  - Location acquisition and tracking
  - Location sharing in messages
  - Geofencing features
- [ ] **4.2.2** Mapping functionality
  - Simple map display
  - Contact location visualization
  - Route planning for optimal communication
- [ ] **4.2.3** Location-based features
  - Proximity alerts
  - Location-based message routing
  - Emergency location beacons

### Milestone 4.3: Configuration and Management
- [ ] **4.3.1** Settings system
  - User preferences (themes, notifications)
  - Radio parameters (frequency, power)
  - Network configuration
- [ ] **4.3.2** Backup and restore
  - Message backup functionality
  - Contact export/import
  - Configuration backup
- [ ] **4.3.3** Monitoring and diagnostics
  - System health monitoring
  - Error logging and reporting
  - Performance profiling

**Phase 4 Deliverables:**
- Multi-hop mesh networking
- GPS integration and mapping
- Comprehensive settings system
- Backup and diagnostic tools

---

## Phase 5: Testing and Optimization (Weeks 9-10)

### Milestone 5.1: Testing Framework
- [ ] **5.1.1** Unit testing
  - Test individual components
  - Mock radio interfaces for testing
  - Automated test suite
- [ ] **5.1.2** Integration testing
  - End-to-end message flow testing
  - Multi-device communication tests
  - Stress testing under load
- [ ] **5.1.3** Field testing
  - Real-world range testing
  - Interference handling
  - Battery life optimization

### Milestone 5.2: Performance Optimization
- [ ] **5.2.1** Memory optimization
  - Reduce memory footprint
  - Optimize data structures
  - Memory leak detection
- [ ] **5.2.2** Power optimization
  - Reduce CPU usage
  - Optimize radio duty cycles
  - Dynamic power management
- [ ] **5.2.3** User experience optimization
  - Reduce UI latency
  - Improve message delivery reliability
  - Enhance error handling

### Milestone 5.3: Documentation and Packaging
- [ ] **5.3.1** User documentation
  - User manual and quick start guide
  - Troubleshooting guide
  - FAQ and tips
- [ ] **5.3.2** Developer documentation
  - API documentation
  - Protocol specifications
  - Build and deployment instructions
- [ ] **5.3.3** Packaging and distribution
  - Create installation packages
  - Automated build system
  - Release versioning

**Phase 5 Deliverables:**
- Comprehensive test suite
- Optimized performance
- Complete documentation
- Distribution packages

---

## Technical Architecture

### Core Components

#### **include/lora/radio_manager.h**
Main radio communication interface.

```cpp
#pragma once

#include <memory>
#include <thread>
#include <atomic>
#include <chrono>
#include <functional>
#include <RH_RF95.h>
#include <RHReliableDatagram.h>
#include "messaging/message_types.h"
#include "utils/common.h"

enum class RadioMode {
    P2P,
    LORAWAN,
    MESH
};

enum class RadioState {
    IDLE,
    TRANSMITTING,
    RECEIVING,
    ERROR
};

struct RadioConfig {
    float frequency = 915.0;        // MHz
    uint8_t spreadingFactor = 7;
    uint8_t bandwidth = 125;        // kHz
    uint8_t codingRate = 5;
    int8_t txPower = 14;           // dBm
    uint16_t preambleLength = 8;
    bool crc = true;
    uint32_t timeout = 5000;       // ms
};

struct NetworkStats {
    uint32_t packetsSent = 0;
    uint32_t packetsReceived = 0;
    uint32_t packetsLost = 0;
    int16_t lastRSSI = -999;
    float lastSNR = -999.0;
    uint32_t uptime = 0;
    std::chrono::system_clock::time_point lastActivity;
};

class RadioManager {
private:
    std::unique_ptr<RH_RF95> radio;
    std::unique_ptr<RHReliableDatagram> reliableRadio;
    RadioConfig config;
    RadioMode currentMode;
    RadioState currentState;
    NetworkStats stats;
    std::thread rxThread;
    std::atomic<bool> running;
    std::function<void(const Message&)> messageCallback;
    
    // SPI pins for Raspberry Pi
    static constexpr int CS_PIN = 8;
    static constexpr int RST_PIN = 22;
    static constexpr int INT_PIN = 25;
    
public:
    RadioManager();
    ~RadioManager();
    
    // Core functions
    bool initialize(const RadioConfig& cfg = RadioConfig{});
    void shutdown();
    bool isInitialized() const;
    
    // Mode management
    bool setMode(RadioMode mode);
    RadioMode getMode() const { return currentMode; }
    RadioState getState() const { return currentState; }
    
    // Message handling
    bool sendMessage(const Message& msg, uint8_t destinationId);
    bool sendBroadcast(const Message& msg);
    void setMessageCallback(std::function<void(const Message&)> callback);
    
    // Configuration
    bool setFrequency(float freq);
    bool setSpreadingFactor(uint8_t sf);
    bool setBandwidth(uint8_t bw);
    bool setTxPower(int8_t power);
    RadioConfig getConfig() const { return config; }
    
    // Statistics
    NetworkStats getStats() const { return stats; }
    void resetStats();
    
    // Low-level operations
    bool available();
    bool recv(uint8_t* buf, uint8_t* len);
    bool send(const uint8_t* data, uint8_t len);
    
private:
    void rxThreadFunction();
    void handleReceivedData(const uint8_t* data, uint8_t len);
    void updateStats(bool sent, bool received, int16_t rssi = -999, float snr = -999.0);
    bool configureRadio();
    void setState(RadioState state);
};
```

#### **include/messaging/message_types.h**
Message structure definitions.

```cpp
#pragma once

#include <string>
#include <vector>
#include <chrono>
#include <cstdint>

enum class MessageType : uint8_t {
    TEXT = 0x01,
    ACK = 0x02,
    HEARTBEAT = 0x03,
    DISCOVERY = 0x04,
    ROUTE_REQUEST = 0x05,
    ROUTE_REPLY = 0x06,
    EMERGENCY = 0x07,
    FILE_TRANSFER = 0x08,
    LOCATION = 0x09,
    SYSTEM = 0x0A
};

enum class MessagePriority : uint8_t {
    LOW = 0,
    NORMAL = 1,
    HIGH = 2,
    EMERGENCY = 3
};

enum class MessageStatus : uint8_t {
    PENDING = 0,
    SENT = 1,
    DELIVERED = 2,
    READ = 3,
    FAILED = 4
};

struct MessageHeader {
    uint16_t magic = 0x4C52;        // "LR"
    uint8_t version = 0x01;
    MessageType type;
    uint16_t sequenceNumber;
    uint16_t payloadLength;
    uint8_t senderId[16];
    uint8_t recipientId[16];
    uint32_t timestamp;
    uint8_t priority;
    uint8_t ttl = 10;               // Time to live for mesh routing
    uint32_t crc32;
} __attribute__((packed));

struct Message {
    MessageHeader header;
    std::vector<uint8_t> payload;
    MessageStatus status = MessageStatus::PENDING;
    std::chrono::system_clock::time_point createdAt;
    std::chrono::system_clock::time_point lastAttempt;
    uint8_t retryCount = 0;
    int16_t rssi = -999;
    float snr = -999.0;
    
    // Helper functions
    std::string getPayloadAsString() const;
    void setPayloadFromString(const std::string& text);
    size_t getTotalSize() const;
    bool isValid() const;
    uint32_t calculateCRC() const;
    std::vector<uint8_t> serialize() const;
    static Message deserialize(const uint8_t* data, size_t len);
};
```

#### **include/messaging/message_queue.h**
Message queue management with priority and retry logic.

```cpp
#pragma once

#include <queue>
#include <vector>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <atomic>
#include <functional>
#include "message_types.h"

struct MessageComparator {
    bool operator()(const Message& a, const Message& b) const {
        if (a.header.priority != b.header.priority) {
            return a.header.priority < b.header.priority;
        }
        return a.createdAt > b.createdAt;
    }
};

class MessageQueue {
private:
    std::priority_queue<Message, std::vector<Message>, MessageComparator> outboundQueue;
    std::vector<Message> inboundQueue;
    std::vector<Message> pendingAcks;
    std::mutex outboundMutex;
    std::mutex inboundMutex;
    std::mutex ackMutex;
    std::condition_variable queueCondition;
    std::thread processingThread;
    std::atomic<bool> running;
    std::function<bool(const Message&)> sendCallback;
    std::function<void(const Message&)> receiveCallback;
    
    // Configuration
    size_t maxRetries = 3;
    std::chrono::milliseconds retryDelay{1000};
    std::chrono::milliseconds ackTimeout{5000};
    size_t maxQueueSize = 1000;
    
public:
    MessageQueue();
    ~MessageQueue();
    
    // Queue management
    bool start();
    void stop();
    bool isRunning() const { return running; }
    
    // Message operations
    bool enqueue(const Message& msg);
    bool enqueueHighPriority(const Message& msg);
    std::vector<Message> dequeueInbound(size_t maxCount = 10);
    size_t getOutboundSize() const;
    size_t getInboundSize() const;
    
    // Acknowledgment handling
    void handleAck(const Message& ackMsg);
    void handleIncomingMessage(const Message& msg);
    
    // Callbacks
    void setSendCallback(std::function<bool(const Message&)> callback);
    void setReceiveCallback(std::function<void(const Message&)> callback);
    
    // Configuration
    void setMaxRetries(size_t retries) { maxRetries = retries; }
    void setRetryDelay(std::chrono::milliseconds delay) { retryDelay = delay; }
    void setAckTimeout(std::chrono::milliseconds timeout) { ackTimeout = timeout; }
    void setMaxQueueSize(size_t size) { maxQueueSize = size; }
    
    // Statistics
    struct QueueStats {
        size_t totalSent = 0;
        size_t totalReceived = 0;
        size_t totalRetries = 0;
        size_t totalDropped = 0;
        size_t currentOutbound = 0;
        size_t currentInbound = 0;
    } stats;
    
    QueueStats getStats() const { return stats; }
    void resetStats();
    
private:
    void processQueue();
    void processRetries();
    void processAcks();
    bool shouldRetry(const Message& msg) const;
    Message createAckMessage(const Message& originalMsg) const;
    void cleanupOldMessages();
};
```

#### **include/ui/ui_manager.h**
User interface management for both TUI and GUI.

```cpp
#pragma once

#include <memory>
#include <string>
#include <vector>
#include <functional>
#include <ncurses.h>
#include "messaging/message_types.h"
#include "messaging/contact_manager.h"
#include "utils/common.h"

enum class UIMode {
    TUI,
    GUI
};

enum class UIState {
    MAIN_MENU,
    CHAT_VIEW,
    CONTACT_LIST,
    SETTINGS,
    STATUS_VIEW,
    COMPOSE_MESSAGE
};

struct UIConfig {
    UIMode mode = UIMode::TUI;
    bool colorSupport = true;
    int refreshRate = 60;           // Hz
    int messageHistoryLimit = 100;
    bool soundEnabled = true;
    bool notificationsEnabled = true;
};

struct ChatMessage {
    std::string senderId;
    std::string senderName;
    std::string content;
    std::chrono::system_clock::time_point timestamp;
    MessageStatus status;
    bool isOutgoing;
};

class UIManager {
private:
    UIConfig config;
    UIState currentState;
    UIState previousState;
    std::vector<ChatMessage> chatHistory;
    std::string currentContact;
    std::string inputBuffer;
    size_t cursorPosition;
    std::shared_ptr<ContactManager> contactManager;
    
    // ncurses windows
    WINDOW* mainWin;
    WINDOW* chatWin;
    WINDOW* inputWin;
    WINDOW* statusWin;
    WINDOW* contactWin;
    
    // UI dimensions
    int screenHeight, screenWidth;
    int chatHeight, inputHeight, statusHeight;
    
    // Callbacks
    std::function<void(const std::string&, const std::string&)> sendMessageCallback;
    std::function<std::vector<Contact>()> getContactsCallback;
    std::function<NetworkStats()> getStatsCallback;
    
public:
    UIManager();
    ~UIManager();
    
    // Core functions
    bool initialize(const UIConfig& cfg = UIConfig{});
    void shutdown();
    bool isInitialized() const;
    
    // Main loop
    void run();
    void handleInput();
    void render();
    void refresh();
    
    // State management
    void setState(UIState state);
    UIState getState() const { return currentState; }
    void goBack();
    
    // Message handling
    void addMessage(const ChatMessage& msg);
    void updateMessageStatus(const std::string& msgId, MessageStatus status);
    void clearChat();
    
    // Contact management
    void setCurrentContact(const std::string& contactId);
    std::string getCurrentContact() const { return currentContact; }
    
    // Callbacks
    void setSendMessageCallback(std::function<void(const std::string&, const std::string&)> callback);
    void setGetContactsCallback(std::function<std::vector<Contact>()> callback);
    void setGetStatsCallback(std::function<NetworkStats()> callback);
    
    // Configuration
    void setConfig(const UIConfig& cfg);
    UIConfig getConfig() const { return config; }
    
    // Notifications
    void showNotification(const std::string& message, int duration = 3000);
    void showError(const std::string& error);
    void showStatus(const std::string& status);
    
private:
    // TUI specific functions
    void initializeTUI();
    void shutdownTUI();
    void renderTUI();
    void handleTUIInput();
    void renderChatView();
    void renderContactList();
    void renderSettings();
    void renderStatusView();
    void renderMainMenu();
    void renderInputField();
    void renderStatusBar();
    
    // Input handling
    void handleKeyPress(int key);
    void handleEnterKey();
    void handleBackspace();
    void handleArrowKeys(int key);
    void handleTab();
    void handleEscape();
    
    // Helper functions
    void updateDimensions();
    void drawBorder(WINDOW* win);
    void centerText(WINDOW* win, int y, const std::string& text);
    std::string formatTimestamp(const std::chrono::system_clock::time_point& time);
    void scrollChatUp();
    void scrollChatDown();
    void wordWrap(std::vector<std::string>& lines, const std::string& text, int width);
    
    // Color management
    void initializeColors();
    void setColor(WINDOW* win, int colorPair);
    static constexpr int COLOR_PAIR_DEFAULT = 1;
    static constexpr int COLOR_PAIR_HEADER = 2;
    static constexpr int COLOR_PAIR_OUTGOING = 3;
    static constexpr int COLOR_PAIR_INCOMING = 4;
    static constexpr int COLOR_PAIR_STATUS = 5;
    static constexpr int COLOR_PAIR_ERROR = 6;
    static constexpr int COLOR_PAIR_SUCCESS = 7;
};
```

### Protocol Specifications

#### P2P Message Format
```
Header (8 bytes):
- Magic Number (2 bytes): 0x4C52 ("LR")
- Version (1 byte): 0x01
- Message Type (1 byte): TEXT/ACK/HEARTBEAT/DISCOVERY
- Sequence Number (2 bytes)
- Payload Length (2 bytes)

Payload (variable):
- Sender ID (16 bytes)
- Recipient ID (16 bytes)
- Timestamp (4 bytes)
- Message Content (variable)

Footer (4 bytes):
- CRC32 checksum
```

#### LoRaWAN Integration
- Use TTN (The Things Network) for infrastructure
- Implement OTAA (Over-The-Air Activation)
- Custom payload format for message routing
- Downlink for message delivery confirmations

---

## Development Milestones and Timeline

### Week 1-2: Foundation
- [ ] Project setup and environment configuration
- [ ] Basic LoRa communication working
- [ ] Simple command-line interface

### Week 3-4: User Interface
- [ ] TUI implementation complete
- [ ] Display integration working
- [ ] Basic messaging interface

### Week 5-6: Messaging System
- [ ] Message persistence working
- [ ] Contact management functional
- [ ] Basic encryption implemented

### Week 7-8: Advanced Features
- [ ] Mesh networking operational
- [ ] GPS integration complete
- [ ] Settings system functional

### Week 9-10: Polish and Testing
- [ ] Full test suite implemented
- [ ] Performance optimization complete
- [ ] Documentation finished

---

## Git Workflow and Branch Strategy

### Branch Structure
```
main            # Stable releases
├── develop     # Integration branch
├── feature/    # Feature development
│   ├── radio-manager
│   ├── tui-interface
│   ├── messaging-system
│   └── lorawan-integration
├── bugfix/     # Bug fixes
└── hotfix/     # Critical fixes
```

### Commit Message Format
```
type(scope): description

[optional body]

[optional footer]
```

Types: feat, fix, docs, style, refactor, test, chore

### Pull Request Template
```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests pass
- [ ] Integration tests pass
- [ ] Manual testing completed

## Checklist
- [ ] Code follows style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
```

---

## Dependencies and Libraries

### Core Libraries

#### **RadioHead Library**
LoRa radio communication library.

**Installation:**
```bash
cd ~/
git clone https://github.com/PaulStoffregen/RadioHead.git
# Copy to project libs directory
cp -r RadioHead ./lora_phone/libs/
```

**Key Classes Used:**
- `RH_RF95` - SX127x radio driver
- `RHReliableDatagram` - Reliable datagram protocol
- `RHRouter` - Mesh networking support
- `RHMesh` - Mesh routing protocol

**Integration in CMakeLists.txt:**
```cmake
# RadioHead sources
file(GLOB RADIOHEAD_SOURCES "${CMAKE_SOURCE_DIR}/libs/RadioHead/*.cpp")
list(APPEND SOURCES ${RADIOHEAD_SOURCES})

# Include RadioHead headers
include_directories(${CMAKE_SOURCE_DIR}/libs/RadioHead)
```

#### **WiringPi Library**
GPIO and SPI access for Raspberry Pi.

**Installation:**
```bash
sudo apt update
sudo apt install wiringpi libwiringpi-dev
```

**Usage:**
```cpp
#include <wiringPi.h>
#include <wiringPiSPI.h>

// Initialize in radio_manager.cpp
wiringPiSetup();
wiringPiSPISetup(0, 1000000);  // SPI channel 0, 1MHz
```

#### **SQLite3 Library**
Message and contact storage.

**Installation:**
```bash
# Download SQLite amalgamation
wget https://www.sqlite.org/2024/sqlite-amalgamation-3460000.zip
unzip sqlite-amalgamation-3460000.zip
cp sqlite-amalgamation-3460000/sqlite3.* ./libs/sqlite/
```

**Database Schema:**
```sql
-- messages table
CREATE TABLE messages (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    sender_id TEXT NOT NULL,
    recipient_id TEXT NOT NULL,
    content TEXT NOT NULL,
    timestamp INTEGER NOT NULL,
    status INTEGER DEFAULT 0,
    message_type INTEGER DEFAULT 1,
    rssi INTEGER DEFAULT -999,
    snr REAL DEFAULT -999.0
);

-- contacts table
CREATE TABLE contacts (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    last_seen INTEGER,
    is_favorite INTEGER DEFAULT 0,
    is_blocked INTEGER DEFAULT 0
);

-- device_info table
CREATE TABLE device_info (
    key TEXT PRIMARY KEY,
    value TEXT
);
```

#### **OpenSSL Library**
Cryptography and encryption.

**Installation:**
```bash
sudo apt install libssl-dev
```

**Usage Example:**
```cpp
#include <openssl/evp.h>
#include <openssl/aes.h>
#include <openssl/rand.h>

// AES-256-GCM encryption
class MessageEncryption {
private:
    std::vector<uint8_t> key;
    
public:
    std::vector<uint8_t> encrypt(const std::vector<uint8_t>& plaintext);
    std::vector<uint8_t> decrypt(const std::vector<uint8_t>& ciphertext);
    bool generateKey();
    bool deriveKey(const std::string& password);
};
```

#### **nlohmann/json Library**
JSON configuration file parsing.

**Installation:**
```bash
cd libs/
wget https://github.com/nlohmann/json/releases/download/v3.11.3/json.hpp
mkdir -p json/include/nlohmann/
mv json.hpp json/include/nlohmann/
```

**Configuration File Format:**
```json
{
  "radio": {
    "frequency": 915.0,
    "spreading_factor": 7,
    "bandwidth": 125,
    "coding_rate": 5,
    "tx_power": 14,
    "enable_crc": true
  },
  "network": {
    "device_id": "LORA_PHONE_001",
    "mesh_enabled": true,
    "auto_discovery": true,
    "heartbeat_interval": 30000
  },
  "ui": {
    "theme": "dark",
    "color_enabled": true,
    "refresh_rate": 60,
    "message_history_limit": 1000
  },
  "security": {
    "encryption_enabled": true,
    "key_exchange_enabled": true,
    "auto_key_rotation": true
  }
}
```

### UI Libraries

#### **ncurses Library**
Terminal user interface.

**Installation:**
```bash
sudo apt install libncurses5-dev libncurses5
```

**Key Functions Used:**
```cpp
// Window management
WINDOW* newwin(int height, int width, int y, int x);
void delwin(WINDOW* win);
void refresh();
void wrefresh(WINDOW* win);

// Input handling
int getch();
int wgetch(WINDOW* win);
void nodelay(WINDOW* win, bool bf);
void keypad(WINDOW* win, bool bf);

// Output
void mvwprintw(WINDOW* win, int y, int x, const char* fmt, ...);
void wattron(WINDOW* win, int attrs);
void wattroff(WINDOW* win, int attrs);

// Colors
void start_color();
void init_pair(short pair, short f, short b);
void wcolor_set(WINDOW* win, short color_pair_number, void* opts);
```

#### **SDL2 Library (Optional)**
Graphics and input handling for GUI mode.

**Installation:**
```bash
sudo apt install libsdl2-dev libsdl2-ttf-dev libsdl2-image-dev
```

**Basic Setup:**
```cpp
#include <SDL2/SDL.h>
#include <SDL2/SDL_ttf.h>

class GUIManager {
private:
    SDL_Window* window;
    SDL_Renderer* renderer;
    TTF_Font* font;
    
public:
    bool initialize();
    void render();
    void handleEvents();
    void cleanup();
};
```

### Utility Libraries

#### **Custom Logger (utils/logger.h)**
Lightweight logging framework.

**Implementation:**
```cpp
#pragma once

#include <string>
#include <fstream>
#include <mutex>
#include <memory>

enum class LogLevel {
    DEBUG = 0,
    INFO = 1,
    WARNING = 2,
    ERROR = 3,
    CRITICAL = 4
};

class Logger {
private:
    std::ofstream logFile;
    LogLevel minLevel;
    std::mutex logMutex;
    static std::shared_ptr<Logger> instance;
    
public:
    static std::shared_ptr<Logger> getInstance();
    bool initialize(const std::string& filename, LogLevel level = LogLevel::INFO);
    void log(LogLevel level, const std::string& message);
    void debug(const std::string& message);
    void info(const std::string& message);
    void warning(const std::string& message);
    void error(const std::string& message);
    void critical(const std::string& message);
};

// Macros for easy logging
#define LOG_DEBUG(msg) Logger::getInstance()->debug(msg)
#define LOG_INFO(msg) Logger::getInstance()->info(msg)
#define LOG_WARNING(msg) Logger::getInstance()->warning(msg)
#define LOG_ERROR(msg) Logger::getInstance()->error(msg)
#define LOG_CRITICAL(msg) Logger::getInstance()->critical(msg)
```

#### **Configuration Manager (utils/config.h)**
JSON configuration management.

**Implementation:**
```cpp
#pragma once

#include <string>
#include <nlohmann/json.hpp>

class ConfigManager {
private:
    nlohmann::json config;
    std::string configPath;
    
public:
    bool loadConfig(const std::string& path);
    bool saveConfig();
    bool saveConfig(const std::string& path);
    
    template<typename T>
    T get(const std::string& key, const T& defaultValue = T{}) const;
    
    template<typename T>
    void set(const std::string& key, const T& value);
    
    bool has(const std::string& key) const;
    void remove(const std::string& key);
    
    // Specific getters for common config values
    float getFrequency() const;
    uint8_t getSpreadingFactor() const;
    uint8_t getBandwidth() const;
    int8_t getTxPower() const;
    std::string getDeviceId() const;
    bool isMeshEnabled() const;
    bool isEncryptionEnabled() const;
};
```

### Build System

#### **CMake Configuration**
Cross-platform build system with automatic dependency detection.

**Complete CMakeLists.txt:**
```cmake
cmake_minimum_required(VERSION 3.16)
project(lora_phone VERSION 1.0.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Build type
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE Release)
endif()

# Compiler flags
set(CMAKE_CXX_FLAGS_DEBUG "-g -O0 -DDEBUG")
set(CMAKE_CXX_FLAGS_RELEASE "-O2 -DNDEBUG")

# Platform detection
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_definitions(-D__linux__)
    if(CMAKE_SYSTEM_PROCESSOR MATCHES "arm")
        add_definitions(-DRASPBERRY_PI)
    endif()
endif()

# Find required packages
find_package(PkgConfig REQUIRED)
find_package(Threads REQUIRED)
find_package(OpenSSL REQUIRED)

# Check for ncurses
pkg_check_modules(NCURSES REQUIRED ncurses)

# Optional: SDL2 for GUI
find_package(SDL2 QUIET)
find_package(SDL2_ttf QUIET)
if(SDL2_FOUND AND SDL2_ttf_FOUND)
    add_definitions(-DSDL2_AVAILABLE)
endif()

# Include directories
include_directories(
    ${CMAKE_SOURCE_DIR}/include
    ${CMAKE_SOURCE_DIR}/libs/RadioHead
    ${CMAKE_SOURCE_DIR}/libs/json/include
    ${CMAKE_SOURCE_DIR}/libs/sqlite
    ${NCURSES_INCLUDE_DIRS}
)

# Source files
file(GLOB_RECURSE SOURCES
    "src/*.cpp"
    "libs/sqlite/sqlite3.c"
)

# RadioHead sources
file(GLOB RADIOHEAD_SOURCES "${CMAKE_SOURCE_DIR}/libs/RadioHead/*.cpp")
list(APPEND SOURCES ${RADIOHEAD_SOURCES})

# Create executable
add_executable(${PROJECT_NAME} ${SOURCES})

# Link libraries
target_link_libraries(${PROJECT_NAME}
    ${NCURSES_LIBRARIES}
    ${CMAKE_THREAD_LIBS_INIT}
    OpenSSL::SSL
    OpenSSL::Crypto
    rt
    dl
)

# Platform-specific libraries
if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    target_link_libraries(${PROJECT_NAME} wiringPi)
endif()

# Optional SDL2 linking
if(SDL2_FOUND AND SDL2_ttf_FOUND)
    target_link_libraries(${PROJECT_NAME} SDL2::SDL2 SDL2_ttf::SDL2_ttf)
endif()

# Compiler warnings
target_compile_options(${PROJECT_NAME} PRIVATE
    -Wall -Wextra -Wpedantic
    -Wno-unused-parameter
    -Wno-unused-variable
)

# Install targets
install(TARGETS ${PROJECT_NAME} DESTINATION bin)
install(DIRECTORY config/ DESTINATION etc/${PROJECT_NAME})
install(FILES README.md DESTINATION share/doc/${PROJECT_NAME})

# Create directories
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/data)
file(MAKE_DIRECTORY ${CMAKE_BINARY_DIR}/data/logs)

# Copy config files
configure_file(${CMAKE_SOURCE_DIR}/config/default_config.json 
               ${CMAKE_BINARY_DIR}/config/default_config.json COPYONLY)

# Testing
enable_testing()
add_subdirectory(tests)

# Package configuration
set(CPACK_PACKAGE_NAME "lora-phone")
set(CPACK_PACKAGE_VERSION ${PROJECT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION "LoRa Communication Device")
set(CPACK_PACKAGE_CONTACT "your.email@example.com")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libncurses5, libssl1.1, libwiringpi2")
include(CPack)
```

#### **Build Scripts**

**scripts/build.sh:**
```bash
#!/bin/bash
set -e

# Create build directory
mkdir -p build
cd build

# Configure
cmake .. -DCMAKE_BUILD_TYPE=Release

# Build
make -j$(nproc)

# Optional: Run tests
if [ "$1" = "test" ]; then
    make test
fi

echo "Build completed successfully!"
```

**scripts/setup_env.sh:**
```bash
#!/bin/bash
set -e

echo "Setting up LoRa Phone development environment..."

# Update system
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y \
    build-essential \
    cmake \
    git \
    libncurses5-dev \
    libssl-dev \
    wiringpi \
    libwiringpi-dev \
    pkg-config

# Create project structure
mkdir -p libs/{RadioHead,json/include/nlohmann,sqlite}
mkdir -p {config,data/logs,tests/{unit,integration,mocks}}

# Download RadioHead
cd libs/
git clone https://github.com/PaulStoffregen/RadioHead.git

# Download nlohmann/json
cd json/include/nlohmann/
wget https://github.com/nlohmann/json/releases/download/v3.11.3/json.hpp

# Download SQLite amalgamation
cd ../../../sqlite/
wget https://www.sqlite.org/2024/sqlite-amalgamation-3460000.zip
unzip sqlite-amalgamation-3460000.zip
mv sqlite-amalgamation-3460000/* .
rm -rf sqlite-amalgamation-3460000*

echo "Environment setup completed!"
```

#### **Package Dependencies**

**debian/control:**
```
Source: lora-phone
Section: comm
Priority: optional
Maintainer: Your Name <your.email@example.com>
Build-Depends: debhelper (>= 9), cmake, libncurses5-dev, libssl-dev, libwiringpi-dev
Standards-Version: 3.9.8

Package: lora-phone
Architecture: armhf
Depends: ${shlibs:Depends}, ${misc:Depends}, libncurses5, libssl1.1, libwiringpi2
Description: LoRa Communication Device
 A Raspberry Pi-based LoRa communication device that functions like a phone
 with automatic fallback between P2P LoRa and LoRaWAN modes.
```

**requirements.txt (for documentation):**
```
# Development dependencies
cmake>=3.16
gcc>=7.5
g++>=7.5

# Runtime dependencies
libncurses5
libssl1.1
libwiringpi2

# Optional dependencies
libsdl2-dev
libsdl2-ttf-dev
```

---

## Testing Strategy

### Unit Testing
- Component-level testing for each class
- Mock objects for radio hardware
- Code coverage targets: >80%

### Integration Testing
- End-to-end message flow testing
- Multi-device communication scenarios
- Network failure simulation

### Field Testing
- Range and performance testing
- Interference handling
- Battery life validation
- Real-world usage scenarios

### Automated Testing
- Continuous integration with GitHub Actions
- Automated builds for multiple targets
- Performance regression testing

---

## Risk Management

### Technical Risks
1. **Radio Hardware Compatibility**
   - Mitigation: Thorough hardware testing early
   - Fallback: Support multiple radio modules

2. **Power Consumption**
   - Mitigation: Implement aggressive power management
   - Fallback: External battery solutions

3. **Range Limitations**
   - Mitigation: Mesh networking implementation
   - Fallback: LoRaWAN infrastructure fallback

### Development Risks
1. **Complexity Creep**
   - Mitigation: Strict milestone adherence
   - Fallback: Feature prioritization

2. **Library Dependencies**
   - Mitigation: Vendor libraries when possible
   - Fallback: Custom implementations

---

## Success Criteria

### Minimum Viable Product (MVP)
- [ ] P2P LoRa communication working
- [ ] Basic TUI interface
- [ ] Message send/receive functionality
- [ ] Contact management
- [ ] Battery operation (4+ hours)

### Full Feature Set
- [ ] Automatic LoRaWAN fallback
- [ ] Mesh networking (3+ hops)
- [ ] End-to-end encryption
- [ ] GPS integration
- [ ] 24+ hour battery life
- [ ] Production-ready packaging

### Performance Targets
- Message delivery: >95% success rate
- Range: 1-5km P2P, unlimited LoRaWAN
- Battery life: 24+ hours typical use
- UI response time: <100ms
- Boot time: <30 seconds

---

## Resources and References

### Documentation
- [RadioHead Library Documentation](http://www.airspayce.com/mikem/arduino/RadioHead/)
- [LoRaWAN Specification](https://lora-alliance.org/resource_hub/lorawan-specification-v1-1/)
- [The Things Network Documentation](https://www.thethingsnetwork.org/docs/)
- [Raspberry Pi GPIO Documentation](https://www.raspberrypi.org/documentation/usage/gpio/)

### Development Tools
- [Visual Studio Code](https://code.visualstudio.com/)
- [CMake](https://cmake.org/)
- [Git](https://git-scm.com/)
- [GitHub](https://github.com/)

### Hardware Resources
- [Pi Supply LoRa HAT Documentation](https://www.pi-supply.com/product/iot-lora-node-phat-for-raspberry-pi/)
- [SX127x Datasheet](https://www.semtech.com/products/wireless-rf/lora-core/sx1276)
- [Raspberry Pi 3B+ Specifications](https://www.raspberrypi.org/products/raspberry-pi-3-model-b-plus/)

---

## Progress Tracking

Use this section to track your progress through the development phases:

### Phase 1: Core Radio Communication
- [ ] Milestone 1.1: Basic LoRa P2P Communication
- [ ] Milestone 1.2: Message Queue and Reliability  
- [ ] Milestone 1.3: LoRaWAN Integration Setup

### Phase 2: User Interface Development
- [ ] Milestone 2.1: Terminal User Interface (TUI)
- [ ] Milestone 2.2: Graphical User Interface (GUI)
- [ ] Milestone 2.3: Display Driver Integration

### Phase 3: Messaging System
- [ ] Milestone 3.1: Message Management
- [ ] Milestone 3.2: Contact Management
- [ ] Milestone 3.3: Encryption and Security

### Phase 4: Advanced Features
- [ ] Milestone 4.1: Network Features
- [ ] Milestone 4.2: Location Services
- [ ] Milestone 4.3: Configuration and Management

### Phase 5: Testing and Optimization
- [ ] Milestone 5.1: Testing Framework
- [ ] Milestone 5.2: Performance Optimization
- [ ] Milestone 5.3: Documentation and Packaging

---

## Notes and Ideas

Use this space to capture ideas, lessons learned, and important notes during development:

### Development Notes
- 

### Performance Observations
- 

### Feature Ideas
- 

### Bug Reports
- 

---

*Last Updated: [Date]*
*Version: 1.0*