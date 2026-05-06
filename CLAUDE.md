Đóng vai trò: Bạn là một chuyên gia phần mềm với Qt/C++ và QML (Qt 6).

Mục tiêu: Viết mã nguồn tạo ra ứng dụng GUI (IoT Gateway Dashboard) quản lý mạng BLE Mesh. Ứng dụng dùng QML cho UI (Front-end) và C++ cho xử lý logic (Controller).

Cơ chế Giao tiếp (IPC - D-Bus):

Ứng dụng giao tiếp với một Backend Daemon thông qua D-Bus session bus.

Tôi đã có sẵn class DbusService (hỗ trợ callMethod, subscribeSignal, emitSignal) và thư viện các DTO (ví dụ SensorDto, ActuatorDto, EventLogDto, DeviceStatusDto có sẵn hàm toJson(), fromJson()). Dữ liệu truyền qua D-Bus luôn là chuỗi JSON.

Thiết kế UI: Sử dụng QtQuick.Controls.Material (Dark/Light mode). Giao diện tối giản, hiện đại.

Cấu trúc giao diện chung (Main Window):

Top Status Bar: Hiển thị icon trạng thái kết nối: Gateway Daemon (Online/Offline), MQTT Broker, Cloud Server.

Main Navigation: Dùng TabBar và SwipeView (hoặc StackLayout) điều hướng 7 Tabs.

Loading State: Có hiệu ứng Loading toàn màn hình hoặc từng tab khi ứng dụng mới mở lên để chờ gọi D-Bus lấy data.

Chi tiết yêu cầu cho từng Tab:

Tab Home (Network Topology): Hiển thị tổng số lượng ESP32. Danh sách thiết bị (Grid/List kèm Tên, Unicast ID, Node Type). Click vào thiết bị mở Popup cấu trúc (Elements, Models) và cấu hình.

Tab Actuator: Hiển thị tổng số lượng Actuator. Giao diện danh sách dạng Card (trạng thái On/Off, mức độ, có Switch/Slider điều khiển). Khi thao tác, gọi D-Bus method SendActuatorCmd.

Tab Sensor: Hiển thị số lượng/phân loại cảm biến. Dùng QtCharts vẽ biểu đồ Line Chart theo thời gian thực (nhận data từ signal SensorData).

Tab Control (Automation): Giao diện hiển thị logic điều khiển (Actuator [X] thuộc Node [A] đang phụ thuộc Sensor [Y] thuộc Node [B]).

Tab Provision:

Form nhập UUID. Nút "Start/Stop Provisioning" (Gọi D-Bus SendProvisionStart / SendProvisionStop).

Khu vực RPR (Remote Provisioning): ComboBox chọn thiết bị có RPR Server. Ô thiết lập Chu kỳ quét (Scan Interval). Nút "Start RPR Scan" (Gọi D-Bus RprScanStart).

Tab History: Bảng log sự kiện (Thời gian, Info/Warn/Error, Node ID, Nội dung). Nhận realtime từ signal MeshEvent.

Tab Update Firmware: Màn hình "Coming Soon" với icon disabled.

Yêu cầu về Code, Cấu trúc C++ & QML:

State Restoration (Khởi động): Class GatewayController (C++) trong quá trình khởi tạo phải gọi DbusService::callMethod (ví dụ gọi method GetAllDevices, GetHistoryLogs từ Backend) để nạp Base Data.

Real-time Updates: GatewayController phải sử dụng DbusService::subscribeSignal để lắng nghe các tín hiệu như DeviceStatus, MeshEvent, SensorData, ActuatorAck. Dùng các class DTO (như SensorDto::fromJson) để parse payload JSON và cập nhật lên UI.

QAbstractListModel: Tạo các class kế thừa QAbstractListModel (C++) cho các danh sách động (Nodes, Actuators, History). Controller sẽ đắp data từ D-Bus vào các model này để QML tự động render.

QML & C++ Bridge: Chia nhỏ UI thành các file .qml (HomeTab.qml, ProvisionTab.qml...). Khai báo Q_PROPERTY, Q_INVOKABLE và signals trong C++ để QML dễ dàng thao tác.


#ifndef IPC_DTO_H
#define IPC_DTO_H

#include <QString>
#include <QJsonDocument>
#include <QJsonObject>
#include <QJsonArray>
#include <stdexcept>
#include "ipc_message.h"   // FIX: thêm include để dùng IpcMessage trong fromJson/toIpcMessage

// ── Helper nội bộ: trích JSON string từ IpcMessage ────────────
inline QString ipcToJson(const IpcMessage& msg)
{
    return msg.payload.value(0).toString();
}

// ── SensorDto ─────────────────────────────────────────────────
struct SensorDto {
    QString node_id;
    int     sensor_id   = 0;
    double  temperature = 0.0;
    double  humidity    = 0.0;
    double  lux         = 0.0;
    int     motion      = 0;
    int     battery     = 0;
    int     status      = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]     = node_id;
        obj["sensor_id"]   = sensor_id;
        obj["temperature"] = temperature;
        obj["humidity"]    = humidity;
        obj["lux"]         = lux;
        obj["motion"]      = motion;
        obj["battery"]     = battery;
        obj["status"]      = status;
        return obj;
    }

    // FIX: thêm toIpcMessage()
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static SensorDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        SensorDto s;
        s.node_id     = obj.value("node_id").toString();
        s.sensor_id   = obj.value("sensor_id").toInt();
        s.temperature = obj.value("temperature").toDouble();
        s.humidity    = obj.value("humidity").toDouble();
        s.lux         = obj.value("lux").toDouble();
        s.motion      = obj.value("motion").toInt();
        s.battery     = obj.value("battery").toInt();
        s.status      = obj.value("status").toInt();
        return s;
    }

    // FIX: overload nhận IpcMessage
    static SensorDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── ActuatorDto ────────────────────────────────────────────────
struct ActuatorDto {
    QString node_id;
    double  setpoint    = 0.0;
    bool    onoff       = false;
    int     actuator_id = 0;
    QString status      = "pending";

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]     = node_id;
        obj["setpoint"]    = setpoint;
        obj["onoff"]       = onoff;
        obj["actuator_id"] = actuator_id;
        obj["status"]      = status;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static ActuatorDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        ActuatorDto a;
        a.node_id     = obj.value("node_id").toString();
        a.setpoint    = obj.value("setpoint").toDouble();
        a.onoff       = obj.value("onoff").toBool();
        a.actuator_id = obj.value("actuator_id").toInt();
        a.status      = obj.value("status").toString("pending");
        return a;
    }

    static ActuatorDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

struct HeartbeatPubDto {
    QString  node_id;
    uint16_t dst    = 0;
    uint8_t  period = 0;
    uint8_t  ttl    = 7;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"] = node_id;
        obj["dst"]     = dst;
        obj["period"]  = period;
        obj["ttl"]     = ttl;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};

struct HeartbeatSubDto {
    QString  node_id;
    uint16_t group_addr = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]    = node_id;
        obj["group_addr"] = group_addr;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};

struct DeleteNodeDto {
    QString node_id;
    QString uuid_hex;  // có thể để rỗng nếu không biết

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]  = node_id;
        obj["uuid_hex"] = uuid_hex;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};

// ─────────────────────────────────────────────────────────
//  SubscribeGroupDto
// ─────────────────────────────────────────────────────────
struct SubscribeGroupDto {
    QString  node_id;
    uint16_t element_addr = 0;
    uint16_t group_addr   = 0;
    uint16_t model_id     = 0;
    uint16_t company_id   = 0xFFFF;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]      = node_id;
        obj["element_addr"] = element_addr;
        obj["group_addr"]   = group_addr;
        obj["model_id"]     = model_id;
        obj["company_id"]   = company_id;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(
            QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};

// ─────────────────────────────────────────────────────────
//  PublishGroupDto
// ─────────────────────────────────────────────────────────
struct PublishGroupDto {
    QString  node_id;
    uint16_t element_addr = 0;
    uint16_t pub_addr     = 0;
    uint16_t model_id     = 0;
    uint16_t company_id   = 0xFFFF;
    uint8_t  ttl          = 7;
    uint8_t  period       = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]      = node_id;
        obj["element_addr"] = element_addr;
        obj["pub_addr"]     = pub_addr;
        obj["model_id"]     = model_id;
        obj["company_id"]   = company_id;
        obj["ttl"]          = ttl;
        obj["period"]       = period;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(
            QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};

// ─────────────────────────────────────────────────────────
//  RprScanStartDto
// ─────────────────────────────────────────────────────────
struct RprScanStartDto {
    QString node_id;
    QString uuid;
    uint8_t limit          = 10;
    uint8_t timeout        = 10;
    bool    uuid_filter_en = false;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]        = node_id;
        obj["uuid"]           = uuid;
        obj["limit"]          = limit;
        obj["timeout"]        = timeout;
        obj["uuid_filter_en"] = uuid_filter_en;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(
            QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};

// ─────────────────────────────────────────────────────────
//  RprLinkOpenDto
// ─────────────────────────────────────────────────────────
struct RprLinkOpenDto {
    QString node_id;
    QString uuid;
    bool    timeout_en = false;
    uint8_t timeout    = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]    = node_id;
        obj["uuid"]       = uuid;
        obj["timeout_en"] = timeout_en;
        obj["timeout"]    = timeout;
        return obj;
    }
    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(
            QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }
};


// ── ActuatorAckDto ────────────────────────────────────────────
struct ActuatorAckDto {
    QString node_id;
    int     actuator_id      = 0;
    int     present_setpoint = 0;
    int     target_setpoint  = 0;
    bool    success          = false;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]          = node_id;
        obj["actuator_id"]      = actuator_id;
        obj["present_setpoint"] = present_setpoint;
        obj["target_setpoint"]  = target_setpoint;
        obj["success"]          = success;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static ActuatorAckDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        ActuatorAckDto a;
        a.node_id          = obj.value("node_id").toString();
        a.actuator_id      = obj.value("actuator_id").toInt();
        a.present_setpoint = obj.value("present_setpoint").toInt();
        a.target_setpoint  = obj.value("target_setpoint").toInt();
        a.success          = obj.value("success").toBool();
        return a;
    }

    static ActuatorAckDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── DeviceDto (legacy / fallback) ─────────────────────────────
// FIX: thêm struct này — được dùng trong fallback AddNewDevice handler
struct DeviceDto {
    QString node_id;
    int     unicast  = 0;
    QString name;
    QString type;    // "sensor" | "actuator" | "relay"
    QString uuid;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"] = node_id;
        obj["unicast"] = unicast;
        obj["name"]    = name;
        obj["type"]    = type;
        obj["uuid"]    = uuid;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static DeviceDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        DeviceDto d;
        d.node_id = obj.value("node_id").toString();
        d.unicast = obj.value("unicast").toInt();
        d.name    = obj.value("name").toString();
        d.type    = obj.value("type").toString();
        d.uuid    = obj.value("uuid").toString();
        return d;
    }

    static DeviceDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── DeviceProvisionedDto ──────────────────────────────────────
struct DeviceProvisionedDto {
    QString node_id;
    QString uuid;
    int32_t net_idx  = 0;
    int32_t elem_num = 1;
    int32_t unicast  = 0;
    QString node_type;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]  = node_id;
        obj["uuid"]     = uuid;
        obj["net_idx"]  = net_idx;
        obj["elem_num"] = elem_num;
        obj["unicast"]  = unicast;
        obj["node_type"]= node_type;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static DeviceProvisionedDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        DeviceProvisionedDto d;
        d.node_id  = obj.value("node_id").toString();
        d.uuid     = obj.value("uuid").toString();
        d.net_idx  = obj.value("net_idx").toInt();
        d.elem_num = obj.value("elem_num").toInt(1);
        d.unicast  = obj.value("unicast").toInt();
        d.node_type = obj.value("node_type").toString();
        return d;
    }

    static DeviceProvisionedDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── DeviceStatusDto ───────────────────────────────────────────
struct DeviceStatusDto {
    QString node_id;
    bool    is_online = false;
    int     features  = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]   = node_id;
        obj["is_online"] = is_online;
        obj["features"]  = features;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static DeviceStatusDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        DeviceStatusDto d;
        d.node_id   = obj.value("node_id").toString();
        d.is_online = obj.value("is_online").toBool();
        d.features  = obj.value("features").toInt();
        return d;
    }

    static DeviceStatusDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── EventLogDto ────────────────────────────────────────────────
struct EventLogDto {
    QString node_id;
    QString level;
    QString event;
    QString payload_json;
    quint64 timestamp = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"]   = node_id;
        obj["level"]     = level;
        obj["event"]     = event;
        obj["payload"]   = payload_json.isEmpty() ? QString("{}") : payload_json;
        obj["timestamp"] = static_cast<qint64>(timestamp);
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static EventLogDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        EventLogDto e;
        e.node_id      = obj.value("node_id").toString();
        e.level        = obj.value("level").toString("info");
        e.event        = obj.value("event").toString();
        if (obj.contains("payload") && obj["payload"].isObject())
            e.payload_json = QJsonDocument(obj["payload"].toObject()).toJson(QJsonDocument::Compact);
        else
            e.payload_json = obj.value("payload").toString();
        e.timestamp    = static_cast<quint64>(obj.value("timestamp").toDouble(0));
        return e;
    }

    static EventLogDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── NodeQueryDto ───────────────────────────────────────────────
struct NodeQueryDto {
    QString node_id;
    int     limit = 50;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["node_id"] = node_id;
        obj["limit"]   = limit;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static NodeQueryDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        NodeQueryDto q;
        q.node_id = obj.value("node_id").toString();
        q.limit   = obj.value("limit").toInt(50);
        return q;
    }

    static NodeQueryDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── UuidWhitelistDto ──────────────────────────────────────────
struct UuidWhitelistDto {
    QString uuid;
    int     bearer = 0;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["uuid"] = uuid;
        obj["bearer"]         = bearer;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static UuidWhitelistDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        UuidWhitelistDto d;
        d.uuid   = obj.value("uuid_match_hex").toString();
        if (d.uuid.isEmpty()) d.uuid = obj.value("uuid").toString();
        d.bearer = obj.value("bearer").toInt();
        return d;
    }

    static UuidWhitelistDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

// ── FirmwareDto ───────────────────────────────────────────────
struct FirmwareDto {
    int     unicast         = 0;
    QString node_type;
    QString current_version;
    QString latest_version;
    bool    is_online = false;

    QJsonObject toJson() const {
        QJsonObject obj;
        obj["unicast"]         = unicast;
        obj["node_type"]       = node_type;
        obj["current_version"] = current_version;
        obj["latest_version"]  = latest_version;
        obj["is_online"]       = is_online;
        return obj;
    }

    IpcMessage toIpcMessage() const {
        IpcMessage msg;
        msg.payload << QString::fromUtf8(QJsonDocument(toJson()).toJson(QJsonDocument::Compact));
        return msg;
    }

    static FirmwareDto fromJson(const QString& json) {
        QJsonObject obj = QJsonDocument::fromJson(json.toUtf8()).object();
        FirmwareDto f;
        f.unicast         = obj.value("unicast").toInt();
        f.node_type       = obj.value("node_type").toString();
        f.current_version = obj.value("current_version").toString();
        f.latest_version  = obj.value("latest_version").toString();
        f.is_online       = obj.value("is_online").toBool();
        return f;
    }

    static FirmwareDto fromJson(const IpcMessage& msg) {
        return fromJson(ipcToJson(msg));
    }
};

#endif // IPC_DTO_H
#ifndef IPC_BRIDGE_H
#define IPC_BRIDGE_H
//checked
#include <QObject>
#include <QString>
#include <functional>
#include <unordered_map>
#include "ipc_message.h"

using BridgeHandler = std::function<void(const IpcMessage&)>;

// IpcBridge: thread-safe dispatch GLib→Qt thread không còn cần thiết
// vì QDBus tự dispatch trên Qt main thread qua QueuedConnection.
// Giữ lại class để Controller code không cần sửa nhiều.
class IpcBridge : public QObject {
    Q_OBJECT
public:
    explicit IpcBridge(QObject* parent = nullptr) : QObject(parent) {
        connect(this, &IpcBridge::_posted,
                this, &IpcBridge::_dispatch,
                Qt::QueuedConnection);
    }

    void on(const QString& signalName, BridgeHandler handler) {
        m_handlers[signalName] = std::move(handler);
    }

    // Gọi từ DbusService::onDbusSignal (đã ở main thread)
    void dispatch(const QString& signalName, const IpcMessage& msg) {
        emit _posted(signalName, msg);
    }

signals:
    void _posted(const QString& signalName, const IpcMessage& msg);

private slots:
    void _dispatch(const QString& signalName, const IpcMessage& msg) {
        auto it = m_handlers.find(signalName);
        if (it != m_handlers.end())
            it->second(msg);
    }

private:
    std::unordered_map<QString, BridgeHandler> m_handlers;
};

#endif
#ifndef IPC_MESSAGE_H
#define IPC_MESSAGE_H
//checked
#include <QVariant>
#include <QString>

// Thay GVariant* bằng QVariantList — Qt-native, không cần GLib
struct IpcMessage {
    QVariantList payload;   // mỗi D-Bus argument là một QVariant
    bool isEmpty() const { return payload.isEmpty(); }
};

#endif
    #ifndef UIPC_TRANSLATOR_H
    #define UIPC_TRANSLATOR_H

    #include <functional>          // FIX: cần cho std::function
    #include <QDateTime>
    #include <QDebug>
    #include <QJsonDocument>
    #include <QJsonObject>

    #include "ipc_dto.h"
    #include "ipc_message.h"
    #include "coremodels.h"
    #include "models.h"

    class UiTranslator {
    public:
        // FIX: khai báo static member — được định nghĩa trong controller.cpp
        static std::function<void(quint16)> s_on_new_device;

        // ── Incoming: DTO → model ─────────────────────────────────

        static void apply_sensor(OverViewModel* ovm, const SensorDto& dto) {
            if (!ovm) return;
            quint16 addr = addr_from_node_id(dto.node_id);
            NodeModel* node = ovm->findNodeByAddr(addr);
            if (!node) {
                qWarning() << "[UiTranslator] apply_sensor: unknown node" << dto.node_id;
                return;
            }
            node->updateSensorData(
                static_cast<float>(dto.temperature),
                static_cast<float>(dto.humidity),
                static_cast<float>(dto.lux));
        }

        static void apply_device_provisioned(OverViewModel* ovm,
                                            ControlModel*  ctrl,
                                            const DeviceProvisionedDto& dto) {
            if (!ovm) return;
            quint16 addr = static_cast<quint16>(dto.unicast);
            NodeModel* node = ovm->findNodeByAddr(addr);
            if (!node) {
                node = new NodeModel(nullptr);
                node->addr     = addr;
                node->name     = dto.node_id.isEmpty()
                                ? QString("0x%1").arg(addr, 4, 16, QChar('0')).toUpper()
                                : dto.node_id;
                node->nodeType = NodeType::SENSOR_NODE;
                node->status   = "provisioned";
                node->lastSeen = static_cast<quint64>(QDateTime::currentMSecsSinceEpoch());
                
                if (dto.node_type == "actuator")
                    node->nodeType = NodeType::ACTUATOR_NODE;
                else if (dto.node_type == "relay")
                    node->nodeType = NodeType::RELAY_NODE;
                else if (dto.node_type == "sensor")
                    node->nodeType = NodeType::SENSOR_NODE;

                ovm->appendNode(node);
                qInfo() << "[UiTranslator] created node addr=0x"
                        << QString::number(addr, 16).toUpper()
                        << "uuid=" << dto.uuid;
                if (ctrl && node->nodeType == NodeType::ACTUATOR_NODE) {
                    QList<NodeModel*> current;
                    for (auto* obj : ctrl->actuators())
                        if (auto* n = qobject_cast<NodeModel*>(obj)) current << n;
                    current << node;
                    ctrl->loadActuators(current);
                }
            } else {
                node->lastSeen = static_cast<quint64>(QDateTime::currentMSecsSinceEpoch());
                emit node->dataChanged();
            }
        }

        // FIX: thêm apply_device cho DeviceDto (legacy fallback)
        static void apply_device(OverViewModel* ovm,
                                ControlModel*  ctrl,
                                const DeviceDto& dto) {
            if (!ovm) return;
            quint16 addr = dto.unicast
                            ? static_cast<quint16>(dto.unicast)
                            : addr_from_node_id(dto.node_id);
            NodeModel* node = ovm->findNodeByAddr(addr);
            if (!node) {
                node = new NodeModel(nullptr);
                node->addr   = addr;
                node->name   = dto.name.isEmpty() ? dto.node_id : dto.name;
                node->status = "offline";
                node->lastSeen = static_cast<quint64>(QDateTime::currentMSecsSinceEpoch());

                QString typeStr = dto.type.toLower();
                if (typeStr == "actuator" || typeStr == "1")
                    node->nodeType = NodeType::ACTUATOR_NODE;
                else if (typeStr == "relay" || typeStr == "2")
                    node->nodeType = NodeType::RELAY_NODE;
                else
                    node->nodeType = NodeType::SENSOR_NODE;

                ovm->appendNode(node);
                qInfo() << "[UiTranslator] apply_device (legacy): addr=0x"
                        << QString::number(addr, 16).toUpper();

                if (ctrl && node->nodeType == NodeType::ACTUATOR_NODE) {
                    QList<NodeModel*> current;
                    for (auto* obj : ctrl->actuators())
                        if (auto* n = qobject_cast<NodeModel*>(obj)) current << n;
                    current << node;
                    ctrl->loadActuators(current);
                }
            } else {
                node->lastSeen = static_cast<quint64>(QDateTime::currentMSecsSinceEpoch());
                emit node->dataChanged();
            }
        }

        static void apply_status(OverViewModel* ovm, const DeviceStatusDto& dto) {
            if (!ovm) return;
            quint16 addr    = addr_from_node_id(dto.node_id);
            NodeModel* node = ovm->findNodeByAddr(addr);
            if (!node) return;
            node->status   = dto.is_online ? "active" : "offline";
            node->lastSeen = static_cast<quint64>(QDateTime::currentMSecsSinceEpoch());
            emit node->dataChanged();
        }

        static void apply_actuator_ack(ControlModel* ctrl, const ActuatorAckDto& dto) {
            if (!ctrl) return;
            quint16 addr = addr_from_node_id(dto.node_id);
            ctrl->onAckReceived(addr, dto.success);
        }

        static void apply_event_log(HistoryModel* hist, const EventLogDto& dto) {
            if (!hist) return;
            LogItem log;
            log.timestamp = dto.timestamp > 0
                                ? static_cast<quint32>(dto.timestamp)
                                : static_cast<quint32>(QDateTime::currentSecsSinceEpoch());
            log.nodeaddr = dto.node_id;
            log.level    = dto.level.toUpper();
            log.message  = dto.event + ": " + dto.payload_json;
            hist->onLogReceived(log);
        }

        // FIX: thêm apply_mesh_event — cập nhật trạng thái gateway/bleMesh
        static void apply_mesh_event(OverViewModel* ovm, const QString& jsonStr) {
            if (!ovm || jsonStr.isEmpty()) return;
            QJsonObject obj = QJsonDocument::fromJson(jsonStr.toUtf8()).object();
            QString ev = obj.value("event").toString();
            if (ev.isEmpty()) return;

            if (ev.startsWith("Gateway", Qt::CaseInsensitive)) {
                bool online = ev.contains("Online",    Qt::CaseInsensitive) ||
                            ev.contains("Connected", Qt::CaseInsensitive) ||
                            ev.contains("Up",        Qt::CaseInsensitive);
                ovm->onGatewayStatus(online);
            } else if (ev.startsWith("Ble", Qt::CaseInsensitive)) {
                bool up = ev.contains("Connected", Qt::CaseInsensitive) ||
                        ev.contains("Up",        Qt::CaseInsensitive) ||
                        ev.contains("Ready",     Qt::CaseInsensitive);
                ovm->onBleMeshStatus(up);
            }
            // Prov* events: chỉ log, không thay đổi status
        }

        // ── Outgoing commands ─────────────────────────────────────

        static IpcMessage make_actuator_cmd(quint16 addr, bool on,
                                            float setpoint    = 0,
                                            int   actuator_id = 0) {
            ActuatorDto dto;
            dto.node_id     = node_id_from_addr(addr);
            dto.onoff       = on;
            dto.setpoint    = static_cast<double>(setpoint);
            dto.actuator_id = actuator_id;
            dto.status      = "pending";
            return dto.toIpcMessage();
        }

        static IpcMessage make_node_query(quint16 addr, int limit = 50) {
            NodeQueryDto q;
            q.node_id = node_id_from_addr(addr);
            q.limit   = limit;
            return q.toIpcMessage();
        }

        // ── addr ↔ node_id ────────────────────────────────────────
        static quint16 addr_from_node_id(const QString& node_id) {
            if (node_id.isEmpty()) return 0;
            if (node_id.startsWith("node_", Qt::CaseInsensitive)) {
                bool ok = false;
                quint16 addr = static_cast<quint16>(node_id.mid(5).toUInt(&ok, 16));
                if (ok) return addr;
            }
            bool ok = false;
            quint16 addr = static_cast<quint16>(node_id.toUInt(&ok, 0));
            if (!ok) {
                qWarning() << "[UiTranslator] cannot parse node_id:" << node_id;
                return 0;
            }
            return addr;
        }

        static quint16 addr_from_node_id(const std::string& node_id) {
            return addr_from_node_id(QString::fromStdString(node_id));
        }

        static QString node_id_from_addr(quint16 addr) {
            return QString("node_%1").arg(addr, 4, 16, QChar('0'));
        }

        static std::string node_id_from_addr_std(quint16 addr) {
            return node_id_from_addr(addr).toStdString();
        }
    };

    #endif // UIPC_TRANSLATOR_H


#ifndef DBUS_SERVICE_H
#define DBUS_SERVICE_H

#include <QObject>
#include <QDBusConnection>
#include <QDBusMessage>
#include <QString>
#include <QList>
#include <QMap>
#include <functional>
#include "ipc_message.h"

/**
 * SignalHandler: function(signalName, IpcMessage)
 * IpcMessage.payload chứa toàn bộ dữ liệu dưới dạng JSON string.
 */
using SignalHandler = std::function<void(const QString& signalName, const IpcMessage& msg)>;

class DbusService : public QObject {
    Q_OBJECT
public:
    explicit DbusService(QObject* parent = nullptr);
    ~DbusService() override = default;

    bool init(const QString& serviceName,
              const QString& objectPath,
              const QString& interfaceName);

    void deInit();

    /**
     * Gọi phương thức D-Bus.
     * @param payload  IpcMessage chỉ có field payload là JSON string.
     *                 Sẽ được wrap vào QDBusVariant nếu không rỗng để tương thích với GLib.
     * @param results  Danh sách các IpcMessage trả về, mỗi cái chứa payload là JSON string
     *                 phân giải từ D-Bus reply arguments.
     */
    bool callMethod(const QString& service,
                    const QString& path,
                    const QString& interface,
                    const QString& method,
                    const IpcMessage& payload,
                    QList<IpcMessage>& results);

    /**
     * Đăng ký lắng nghe tín hiệu D-Bus.
     * Khi tín hiệu đến, payload JSON sẽ được trích xuất và truyền cho handler.
     */
    void subscribeSignal(const QString& service,
                         const QString& path,
                         const QString& interface,
                         const QString& signalName,
                         SignalHandler  handler);

    /**
     * Gửi tín hiệu D-Bus với payload là JSON string.
     */
    bool emitSignal(const QString& signalName,
                    const IpcMessage& data);

signals:
    void signalReceived(const QString& signalName, const IpcMessage& msg);

private slots:
    void onDbusSignal(const QDBusMessage& msg);

private:
    /** Trích xuất chuỗi JSON từ tham số đầu tiên của thông điệp D-Bus. */
    static QString extractJsonPayload(const QDBusMessage& msg);

    QDBusConnection m_bus;
    QString         m_objectPath;
    QString         m_interfaceName;
    bool            m_initialized = false;

    QMap<QString, SignalHandler> m_signalHandlers;
};

#endif // DBUS_SERVICE_H

#include "dbus_service.h"
#include <QDBusConnectionInterface>
#include <QDBusVariant>
#include <QDebug>
#include <QJsonDocument>
#include <QJsonObject>

DbusService::DbusService(QObject* parent)
    : QObject(parent)
    , m_bus(QDBusConnection::sessionBus())
{}

bool DbusService::init(const QString& serviceName,
                       const QString& objectPath,
                       const QString& interfaceName)
{
    m_objectPath    = objectPath;
    m_interfaceName = interfaceName;

    if (!m_bus.isConnected()) {
        qCritical() << "[DbusService] Cannot connect to session bus";
        return false;
    }

    auto reply = m_bus.interface()->registerService(
        serviceName,
        QDBusConnectionInterface::ReplaceExistingService,
        QDBusConnectionInterface::AllowReplacement);

    if (!reply.isValid()) {
        qWarning() << "[DbusService] registerService failed:"
                   << reply.error().message();
    }

    m_initialized = true;
    qInfo() << "[DbusService] initialized:" << serviceName << objectPath;
    return true;
}

void DbusService::deInit()
{
    m_signalHandlers.clear();
    m_initialized = false;
}

bool DbusService::callMethod(const QString& service,
                             const QString& path,
                             const QString& interface,
                             const QString& method,
                             const IpcMessage& payload,
                             QList<IpcMessage>& results)
{
    if (!m_bus.isConnected()) return false;

    QDBusMessage call = QDBusMessage::createMethodCall(service, path, interface, method);

    if (!payload.payload.isEmpty()) {
        QDBusVariant wrapped;
        // FIX B1: lấy QString ở index [0] thay vì bọc cả QVariantList
        // payload.payload là QVariantList{QString}, không phải QString
        // GLib dùng write_payload: g_variant_new("(v)", GVariant*[s])
        // → Qt phải gửi QDBusVariant chứa QString, không phải QVariantList
        wrapped.setVariant(payload.payload.value(0));
        call.setArguments(QVariantList() << QVariant::fromValue(wrapped));
    }

    QDBusMessage reply = m_bus.call(call, QDBus::Block, 5000);

    if (reply.type() == QDBusMessage::ErrorMessage) {
        qWarning() << "[DbusService] callMethod error:"
                   << reply.errorName() << reply.errorMessage();
        return false;
    }

    for (const QVariant& arg : reply.arguments()) {
        IpcMessage result;
        if (arg.canConvert<QDBusVariant>()) {
            QDBusVariant dbv = arg.value<QDBusVariant>();
            QVariant inner = dbv.variant();
            if (inner.type() == QVariant::String) {
                result.payload << inner.toString();
            } else {
                result.payload << QString::fromUtf8(
                    QJsonDocument::fromVariant(inner).toJson(QJsonDocument::Compact));
            }
        } else if (arg.type() == QVariant::String) {
            result.payload << arg.toString();
        } else {
            result.payload << QString::fromUtf8(
                QJsonDocument::fromVariant(arg).toJson(QJsonDocument::Compact));
        }
        results.append(result);
    }

    return true;
}

void DbusService::subscribeSignal(const QString& service,
                                  const QString& path,
                                  const QString& interface,
                                  const QString& signalName,
                                  SignalHandler  handler)
{
    m_signalHandlers[signalName] = std::move(handler);

    bool ok = m_bus.connect(
        service, path, interface, signalName,
        this, SLOT(onDbusSignal(QDBusMessage)));

    if (ok)
        qInfo() << "[DbusService] subscribed:" << interface << "::" << signalName;
    else
        qWarning() << "[DbusService] subscribe failed:" << signalName;
}

void DbusService::onDbusSignal(const QDBusMessage& msg)
{
    const QString signalName = msg.member();
    IpcMessage ipcMsg;
    ipcMsg.payload << extractJsonPayload(msg);

    auto it = m_signalHandlers.find(signalName);
    if (it != m_signalHandlers.end())
        it.value()(signalName, ipcMsg);

    emit signalReceived(signalName, ipcMsg);
}

bool DbusService::emitSignal(const QString& signalName,
                             const IpcMessage& data)
{
    if (!m_initialized) return false;

    QDBusMessage signal = QDBusMessage::createSignal(
        m_objectPath, m_interfaceName, signalName);
    signal.setArguments(QVariantList() << data.payload);

    bool ok = m_bus.send(signal);
    if (!ok)
        qWarning() << "[DbusService] emitSignal failed:" << signalName;
    return ok;
}

QString DbusService::extractJsonPayload(const QDBusMessage& msg)
{
    const QVariantList args = msg.arguments();
    if (args.isEmpty()) return QString();

    const QVariant& first = args.first();

    if (first.type() == QVariant::String)
        return first.toString();

    if (first.canConvert<QDBusVariant>()) {
        QDBusVariant dbv = first.value<QDBusVariant>();
        QVariant inner = dbv.variant();
        if (inner.type() == QVariant::String)
            return inner.toString();
        return QString::fromUtf8(
            QJsonDocument::fromVariant(inner).toJson(QJsonDocument::Compact));
    }

    return QString::fromUtf8(
        QJsonDocument::fromVariant(first).toJson(QJsonDocument::Compact));
}

/// Day la file gatewayservice cua phia daemon app, noi chiu trach nhiem phat cac event ra ngoai

#include "gateway_service.hpp"
#include "ipc.h"
#include "ipc_message.h"
#include "ipc_dto.h"

#include "gateway_translator.h"
#include <iostream>
#include <sstream>
#include <iomanip>
#include <cstring>
#include <algorithm>
#include <fstream>


void GatewayService::load_config(const std::string& path)
{
    std::ifstream f(path);
    if (!f.is_open()) {
        std::cerr << "[GW] Cannot open config: " << path << "\n";
        return;
    }
    std::stringstream buf;
    buf << f.rdbuf();
    std::string content = buf.str();

    // ── 1. Load app_keys ──────────────────────────────────────────────
    {
        size_t sec = content.find("\"app_keys\"");
        if (sec != std::string::npos) {
            size_t arr_end   = content.find("]", sec);
            size_t obj_start = content.find("{", sec);
            while (obj_start < arr_end && obj_start != std::string::npos) {
                size_t obj_end = content.find("}", obj_start);
                if (obj_end == std::string::npos || obj_end > arr_end) break;
                std::string block = content.substr(obj_start, obj_end - obj_start + 1);

                AppKeyConfig ak;
                ak.net_idx = jutil::get_int(block, "net_idx", 0);
                ak.app_idx = jutil::get_int(block, "app_idx", 0);
                std::string key_hex = jutil::get_str(block, "key", "");
                if (key_hex.size() >= 32) {
                    for (int i = 0; i < 16; i++)
                        ak.key[i] = (uint8_t)std::stoul(key_hex.substr(i * 2, 2), nullptr, 16);
                }
                app_keys_.push_back(ak);
                obj_start = content.find("{", obj_end + 1);
            }
        }
        std::cout << "[GW] Loaded " << app_keys_.size() << " app keys\n";
    }

    // ── 2. Load model_bind_rules ──────────────────────────────────────
    {
        // FIX Bug 1: đổi "binding_rules" → "model_bind_rules"
        size_t sec = content.find("\"model_bind_rules\"");
        if (sec == std::string::npos) {
            std::cerr << "[GW] No model_bind_rules in config\n";
            return;
        }
        size_t arr_end   = content.find("]", sec);
        size_t obj_start = content.find("{", sec);
        while (obj_start < arr_end && obj_start != std::string::npos) {
            size_t obj_end = content.find("}", obj_start);
            if (obj_end == std::string::npos || obj_end > arr_end) break;
            std::string block = content.substr(obj_start, obj_end - obj_start + 1);

            ModelBindRule rule;
            // FIX Bug 2: gán model_id và company_id vào rule
            rule.model_id   = jutil::get_int(block, "model_id",   0);
            rule.company_id = jutil::get_int(block, "company_id", 0xFFFF);
            rule.app_idx    = jutil::get_int(block, "app_idx",    0);
            rule.node_type  = jutil::get_str(block, "node_type",  "unknown");

            if (rule.model_id != 0)
                bind_rules_.push_back(rule);

            obj_start = content.find("{", obj_end + 1);
        }
        std::cout << "[GW] Loaded " << bind_rules_.size() << " bind rules\n";
    }
}

GatewayService::GatewayService(std::shared_ptr<ISerialPort> serial,
                               std::shared_ptr<IIpc>         ipc)
    : ipc(ipc)
{
    transport  = std::make_shared<ReliableTransport>(serial);
    codec      = std::make_shared<FrameCodec>();
    sender     = std::make_shared<MeshCommandSender>(transport, codec);
    dispatcher = std::make_shared<MeshEventDispatcher>();
}

GatewayService::~GatewayService() { stop(); }


/* ══════════════════════════════════════════════════════════════════════════
   Start / Stop
   ══════════════════════════════════════════════════════════════════════════ */

communication_status_t GatewayService::start()
{
    load_config("mesh_config.json");
    register_handlers();

    transport->set_frame_callback([this](const RawFrame& raw) {
        auto mesh = codec->decode(raw);
        if (!mesh) return;
        {
            std::lock_guard<std::mutex> lk(mesh_mutex);
            mesh_event_queue.push(*mesh);
        }
        mesh_cv.notify_one();
    });

    auto status = transport->start();
    if (status != COMMUNICATION_SUCCESS) {
        std::cerr << "[GW] Can't open serial port\n";
        return status;
    }

    is_running = true;
    worker_thread = std::thread(&GatewayService::worker_loop, this);
    std::cout << "[GW] Started\n";
    return COMMUNICATION_SUCCESS;
}

void GatewayService::stop()
{
    if (!is_running.exchange(false)) return;

    mesh_cv.notify_all();
    transport->stop();

    if (worker_thread.joinable()) worker_thread.join();

    std::cout << "[GW] Stopped\n";
}


/* ══════════════════════════════════════════════════════════════════════════
   register_handlers
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::register_handlers()
{
    using O = OpCode;

    // ── Section 6: Health / Node ─────────────────────────────────────────
    dispatcher->on(O::EVT_NODE_RESET,       [this](const MeshFrame& f){ on_node_reset(f);       });
    dispatcher->on(O::EVT_HEALTH_FAULT,     [this](const MeshFrame& f){ on_health_fault(f);     });
    dispatcher->on(O::EVT_HEALTH_ATTENTION, [this](const MeshFrame& f){ on_health_attention(f); });

    // ── Section 7: Provisioning ──────────────────────────────────────────
    dispatcher->on(O::EVT_PROV_ENABLE_COMP,    [this](const MeshFrame& f){ on_prov_enable_comp(f);    });
    dispatcher->on(O::EVT_PROV_DISABLE_COMP,   [this](const MeshFrame& f){ on_prov_disable_comp(f);   });
    dispatcher->on(O::EVT_RECV_UNPROV_ADV_PKT, [this](const MeshFrame& f){ on_recv_unprov_adv_pkt(f); });
    dispatcher->on(O::EVT_PROV_LINK_OPEN,      [this](const MeshFrame& f){ on_prov_link_open(f);      });
    dispatcher->on(O::EVT_PROV_LINK_CLOSE,     [this](const MeshFrame& f){ on_prov_link_close(f);     });
    dispatcher->on(O::EVT_PROV_COMPLETE,       [this](const MeshFrame& f){ on_prov_complete(f);       });
    dispatcher->on(O::EVT_ADD_UNPROV_DEV_COMP, [this](const MeshFrame& f){ on_add_unprov_dev_comp(f); });
    dispatcher->on(O::EVT_SET_UUID_MATCH_COMP, [this](const MeshFrame& f){ on_set_uuid_match_comp(f); });
    dispatcher->on(O::EVT_SET_NODE_NAME_COMP,  [this](const MeshFrame& f){ on_set_node_name_comp(f);  });
    dispatcher->on(O::EVT_ADD_APP_KEY_COMP,    [this](const MeshFrame& f){ on_add_app_key_comp(f);    });
    dispatcher->on(O::EVT_BIND_APP_KEY_COMP,   [this](const MeshFrame& f){ on_bind_app_key_comp(f);   });
    dispatcher->on(O::EVT_GROUP_ADD_COMP,      [this](const MeshFrame& f){ on_group_add_comp(f);      });
    dispatcher->on(O::EVT_GROUP_REMOVE_COMP,   [this](const MeshFrame& f){ on_group_remove_comp(f);   });

    // ── Section 8: Config Client ─────────────────────────────────────────
    dispatcher->on(O::EVT_MODEL_COMP_DATA_STATUS, [this](const MeshFrame& f){ on_comp_data_status(f);   });
    dispatcher->on(O::EVT_CFG_CLIENT_TIMEOUT,     [this](const MeshFrame& f){ on_cfg_client_timeout(f); });

    // ── Section 9: Generic Client ────────────────────────────────────────
    dispatcher->on(O::EVT_GENERIC_ONOFF_STATUS,   [this](const MeshFrame& f){ on_onoff_status(f);           });
    dispatcher->on(O::EVT_SENSOR_STATUS,          [this](const MeshFrame& f){ on_sensor_status(f);          });
    dispatcher->on(O::EVT_ACTUATOR_STATUS,        [this](const MeshFrame& f){ on_actuator_status(f);        });
    dispatcher->on(O::EVT_GENERIC_CLIENT_TIMEOUT, [this](const MeshFrame& f){ on_generic_client_timeout(f); });

    // ── Section 10: RPR ──────────────────────────────────────────────────
    dispatcher->on(O::EVT_RPR_SCAN_STATUS,    [this](const MeshFrame& f){ on_rpr_scan_status(f);     });
    dispatcher->on(O::EVT_RPR_SCAN_REPORT,    [this](const MeshFrame& f){ on_rpr_scan_report(f);     });
    dispatcher->on(O::EVT_RPR_LINK_STATUS,    [this](const MeshFrame& f){ on_rpr_link_status(f);     });
    dispatcher->on(O::EVT_RPR_LINK_OPEN,      [this](const MeshFrame& f){ on_rpr_link_open_evt(f);   });
    dispatcher->on(O::EVT_RPR_LINK_CLOSE,     [this](const MeshFrame& f){ on_rpr_link_close_evt(f);  });
    dispatcher->on(O::EVT_RPR_LINK_REPORT,    [this](const MeshFrame& f){ on_rpr_link_report(f);     });
    dispatcher->on(O::EVT_RPR_START_PROV_COMP,[this](const MeshFrame& f){ on_rpr_start_prov_comp(f); });
    dispatcher->on(O::EVT_RPR_PROV_COMPLETE,  [this](const MeshFrame& f){ on_rpr_prov_complete(f);   });

    // ── Section 11: Group / Heartbeat ────────────────────────────────────
    dispatcher->on(O::EVT_MODEL_SUBSCRIBE_STATUS,   [this](const MeshFrame& f){ on_model_subscribe_status(f);   });
    dispatcher->on(O::EVT_MODEL_UNSUBSCRIBE_STATUS, [this](const MeshFrame& f){ on_model_unsubscribe_status(f); });
    dispatcher->on(O::EVT_MODEL_PUBLISH_STATUS,     [this](const MeshFrame& f){ on_model_publish_status(f);     });
    dispatcher->on(O::EVT_HEARTBEAT,                [this](const MeshFrame& f){ on_heartbeat(f);                });
    dispatcher->on(O::EVT_HEARTBEAT_PUB_STATUS,     [this](const MeshFrame& f){ on_heartbeat_pub_status(f);     });
    dispatcher->on(O::EVT_HEARTBEAT_SUB_STATUS,     [this](const MeshFrame& f){ on_heartbeat_sub_status(f);     });

    // ── Section 12: Network / Feature State ──────────────────────────────
    dispatcher->on(O::EVT_RELAY_STATUS,           [this](const MeshFrame& f){ on_relay_status(f);           });
    dispatcher->on(O::EVT_PROXY_STATUS,           [this](const MeshFrame& f){ on_proxy_status(f);           });
    dispatcher->on(O::EVT_FRIEND_STATUS,          [this](const MeshFrame& f){ on_friend_status(f);          });
    dispatcher->on(O::EVT_FRIENDSHIP_ESTABLISHED, [this](const MeshFrame& f){ on_friendship_established(f); });
    dispatcher->on(O::EVT_FRIENDSHIP_TERMINATED,  [this](const MeshFrame& f){ on_friendship_terminated(f);  });

//     // ── IPC Methods ───────────────────────────────────────────────────────
    if (!ipc) return;

    /* ── Bật scan provision ──────────────────────────────────────────────── */
    auto& snd = *sender;

    ipc->expose_method("SendProvisionStart", GatewayTranslator::make_provision_start_handler(snd, known_uuids_mutex, known_uuids));
    ipc->expose_method("SendProvisionStop",  GatewayTranslator::make_provision_stop_handler(snd));
    
    ipc->expose_method("SetUuidMatch",       GatewayTranslator::make_set_uuid_match_handler(snd));
    ipc->expose_method("DeleteNode",         GatewayTranslator::make_delete_node_handler(snd, known_uuids_mutex, known_uuids));
    
    ipc->expose_method("SendActuatorCmd",    GatewayTranslator::make_actuator_cmd_handler(snd));
    
    ipc->expose_method("SubscribeToGroup",   GatewayTranslator::make_subscribe_group_handler(snd));
    ipc->expose_method("PublishToGroup",     GatewayTranslator::make_publish_group_handler(snd));
    
    ipc->expose_method("RprScanStart",       GatewayTranslator::make_rpr_scan_start_handler(snd));
    ipc->expose_method("RprLinkOpen",        GatewayTranslator::make_rpr_link_open_handler(snd));
    
    ipc->expose_method("HeartbeatPub",       GatewayTranslator::make_heartbeat_pub_handler(snd));
    ipc->expose_method("HeartbeatSub",       GatewayTranslator::make_heartbeat_sub_handler(snd));
}


/* ══════════════════════════════════════════════════════════════════════════
   Worker loop
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::worker_loop()
{
    while (is_running.load()) {
        MeshFrame frame;
        {
            std::unique_lock<std::mutex> lk(mesh_mutex);
            mesh_cv.wait(lk, [this]{
                return !mesh_event_queue.empty() || !is_running.load();
            });
            if (!is_running.load() && mesh_event_queue.empty()) break;
            frame = mesh_event_queue.front();
            mesh_event_queue.pop();
        }

        dispatcher->dispatch(frame);
    }
}


/* ══════════════════════════════════════════════════════════════════════════
   Section 6 — Health / Node Reset
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_node_reset(const MeshFrame &f)
{
    std::cout << "[GW] Node reset 0x" << std::hex << f.addr << "\n";

    if (!ipc) return;

    DeviceStatusDto dto;
    dto.node_id   = node_id_from_unicast(f.addr);
    dto.is_online = false;
    dto.features  = 0;

    ipc->send(dto_to_ipc(dto), "DeviceStatus");
}

void GatewayService::on_health_fault(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_health_fault_t>(f);
    std::cerr << "[GW] Health fault node=0x" << std::hex << f.addr << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "error";
    evt.event   = "HealthFault";
    if (p) {
        std::ostringstream js;
        js << "{\"fault_array\":["
           << (int)p->fault_array[0] << ","
           << (int)p->fault_array[1] << ","
           << (int)p->fault_array[2] << ","
           << (int)p->fault_array[3] << "]}";
        evt.payload_json = js.str();
    } else {
        evt.payload_json = "{}";
    }
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_health_attention(const MeshFrame& f)
{
    std::cout << "[GW] Health attention node=0x" << std::hex << f.addr << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "warn";
    evt.event        = "HealthAttention";
    evt.payload_json = "{}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


/* ══════════════════════════════════════════════════════════════════════════
   Section 7 — Provisioning events
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_prov_enable_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_status_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] Prov enable " << (ok ? "OK" : "FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = "gateway";
    evt.level        = "info";
    evt.event        = "ProvEnableComp";
    evt.payload_json = std::string("{\"status\":") + (ok ? "0" : "1") + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_prov_disable_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_status_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] Prov disable " << (ok ? "OK" : "FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = "gateway";
    evt.level        = "info";
    evt.event        = "ProvDisableComp";
    evt.payload_json = std::string("{\"status\":") + (ok ? "0" : "1") + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_recv_unprov_adv_pkt(const MeshFrame& f)
{
    if (f.payload.size() < sizeof(mesh_evt_unprov_adv_t)) return;
    auto* p = payload_cast<mesh_evt_unprov_adv_t>(f);
    if (!p) return;

    std::string uuid_hex = bytes_to_hex(p->uuid, 16);
    std::string mac_hex  = bytes_to_hex(p->bd_addr, 6);

    std::cout << "[GW] UnprovAdv UUID=" << uuid_hex
              << " MAC=" << mac_hex
              << " RSSI=" << (int)(int8_t)p->rssi << "\n";

    if (!ipc) return;

    UnprovAdvDto adv_dto;
    adv_dto.uuid     = uuid_hex;
    adv_dto.mac      = mac_hex;
    adv_dto.rssi     = static_cast<int32_t>((int8_t)p->rssi);  // sign-extend
    adv_dto.bearer   = p->bearer;
    adv_dto.oob_info = static_cast<uint8_t>(p->oob_info & 0xFF);
    ipc->send(dto_to_ipc(adv_dto), "UnprovAdv");

    // ── Signal 2: MeshEvent (log) ─────────────────────────────────────────
    EventLogDto evt;
    evt.node_id = "gateway";
    evt.level   = "info";
    evt.event   = "UnprovAdvPkt";
    std::ostringstream js;
    js << "{\"uuid\":\""  << uuid_hex  << "\""
       << ",\"mac\":\""   << mac_hex   << "\""
       << ",\"bearer\":"  << (int)p->bearer
       << ",\"rssi\":"    << (int)(int8_t)p->rssi << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_prov_link_open(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_prov_link_t>(f);
    std::cout << "[GW] Prov link open bearer="
              << (p && p->bearer == 0 ? "PB-ADV" : "PB-GATT") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = "gateway";
    evt.level        = "info";
    evt.event        = "ProvLinkOpen";
    evt.payload_json = std::string("{\"bearer\":") + std::to_string(p ? (int)p->bearer : 0) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_prov_link_close(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_prov_link_t>(f);
    if (!p) return;
    std::cout << "[GW] Prov link close bearer="
              << (p->bearer == 0 ? "PB-ADV" : "PB-GATT")
              << " reason=0x" << std::hex << (int)p->reason << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = "gateway";
    evt.level   = "info";
    evt.event   = "ProvLinkClose";
    std::ostringstream js;
    js << "{\"bearer\":" << (int)p->bearer << ",\"reason\":" << (int)p->reason << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_prov_complete(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_prov_complete_t>(f);
    if (!p) return;

    std::string uuid_hex = bytes_to_hex(p->uuid, 16);
    std::cout << "[GW] ProvComplete node=0x" << std::hex << f.addr
              << " elem_num=" << (int)p->elem_num
              << " net_idx="  << p->net_idx
              << " uuid="     << uuid_hex << "\n";

    {
        std::lock_guard<std::mutex> lk(known_uuids_mutex);
        known_uuids.insert(uuid_hex);
    }

    if (!ipc) return;

    // ── Signal 1: AddNewDevice ────────────────────────────────────────────
    DeviceProvisionedDto dev_dto;
    dev_dto.node_id  = node_id_from_unicast(f.addr);
    dev_dto.uuid     = uuid_hex;
    dev_dto.net_idx  = static_cast<int32_t>(p->net_idx);
    dev_dto.elem_num = static_cast<int32_t>(p->elem_num);
    dev_dto.unicast  = static_cast<int32_t>(f.addr);
    ipc->send(dto_to_ipc(dev_dto), "AddNewDevice");

    // ── Signal 2: MeshEvent (log) ─────────────────────────────────────────
    EventLogDto evt;
    evt.node_id = dev_dto.node_id;
    evt.level   = "info";
    evt.event   = "ProvComplete";
    std::ostringstream js;
    js << "{\"elem_num\":" << (int)p->elem_num
       << ",\"net_idx\":"  << p->net_idx
       << ",\"uuid\":\""   << uuid_hex << "\"}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");

    start_comp_data_pipeline(f.addr, uuid_hex);
}

void GatewayService::start_comp_data_pipeline(uint16_t unicast, const std::string& uuid)
{
    {
        std::lock_guard<std::mutex> lk(prov_mutex_);
        ProvisioningContext ctx;
        ctx.unicast = unicast;
        ctx.uuid    = uuid;
        prov_contexts_[unicast] = ctx;
    }
    sender->get_comp_data(unicast, 0); 
}


void GatewayService::on_add_unprov_dev_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_status_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] AddUnprovDev " << (ok ? "OK" : "FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = "gateway";
    evt.level        = "info";
    evt.event        = "AddUnprovDevComp";
    evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0xFF) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_set_uuid_match_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_status_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] SetUuidMatch " << (ok ? "OK" : "FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = "gateway";
    evt.level        = "info";
    evt.event        = "SetUuidMatchComp";
    evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0xFF) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_set_node_name_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_status_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] SetNodeName node=0x" << std::hex << f.addr
              << (ok ? " OK" : " FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "SetNodeNameComp";
    evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0xFF) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_add_app_key_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_status_t>(f);
    bool ok = p && (p->status_code == 0x00 || p->status_code == 0x04);

    std::cout << "[GW] AddAppKey addr=0x" << std::hex << f.addr
              << (ok ? " OK" : " FAILED")
              << " status=0x" << (p ? (int)p->status_code : 0xFF) << "\n";

    if (!ok) {
        std::lock_guard<std::mutex> lk(prov_mutex_);
        prov_contexts_.erase(f.addr);
        return;
    }

    // Advance key phase
    {
        std::lock_guard<std::mutex> lk(prov_mutex_);
        auto it = prov_contexts_.find(f.addr);
        if (it != prov_contexts_.end())
            it->second.key_add_idx++;
    }

    if (ipc) {
        EventLogDto evt;
        evt.node_id      = node_id_from_unicast(f.addr);
        evt.level        = "info";
        evt.event        = "AddAppKeyComp";
        evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0) + "}";
        ipc->send(dto_to_ipc(evt), "MeshEvent");
    }

    // FIX Bug 4 & 5: dùng advance_provisioning thay vì prov_key_app_idx undefined
    advance_provisioning(f.addr);
}

void GatewayService::on_bind_app_key_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_cfg_model_op_t>(f);
    if (!p) return;

    bool ok = (p->status_code == 0x00);

    if (!ok) {
        std::cerr << "[GW] BindAppKey FAILED model=0x" << std::hex
                  << p->model_id << " status=0x" << (int)p->status_code << "\n";
        // Retry: KHÔNG increment bind_idx → advance_provisioning sẽ gửi lại cùng model
        advance_provisioning(f.addr);
        return;
    }
    std::cout << "[GW] BindAppKey node=0x" << std::hex << f.addr
              << " model=0x" << p->model_id
              << (ok ? " OK" : " FAILED") << "\n";

    if (ipc) {
        EventLogDto evt;
        evt.node_id = node_id_from_unicast(f.addr);
        evt.level   = ok ? "info" : "warn";
        evt.event   = "BindAppKeyComp";
        std::ostringstream js;
        js << "{\"status\":"     << (int)p->status_code
           << ",\"model_id\":"   << (int)p->model_id
           << ",\"company_id\":" << (int)p->company_id
           << ",\"node_ready\":false}";
        evt.payload_json = js.str();
        ipc->send(dto_to_ipc(evt), "MeshEvent");
    }

    // Advance bind_idx rồi gọi advance
    {
        std::lock_guard<std::mutex> lk(prov_mutex_);
        auto it = prov_contexts_.find(f.addr);
        if (it != prov_contexts_.end())
            it->second.bind_idx++;
    }
    advance_provisioning(f.addr);
}

void GatewayService::bind_next(uint16_t unicast)
{
    advance_provisioning(unicast);
}


void GatewayService::on_bind_complete(uint16_t unicast, const std::string& node_type, bool has_rpr_server)
{
    if (has_rpr_server) {
        std::lock_guard<std::mutex> lk(rpr_servers_mutex_);
        rpr_servers_.insert(unicast);
        std::cout << "[GW] Registered RPR server: 0x" 
                  << std::hex << unicast << "\n";
    }

    std::cout << "[GW] Node 0x" << std::hex << unicast
              << " fully configured type=" << node_type << "\n";

    if (!ipc) return;

    // Signal 1: Cập nhật AddNewDevice với node_type đã biết
    DeviceProvisionedDto dev_dto;
    dev_dto.node_id   = node_id_from_unicast(unicast);
    dev_dto.unicast   = static_cast<int32_t>(unicast);
    dev_dto.node_type = node_type;
    dev_dto.has_rpr_server = has_rpr_server;
    ipc->send(dto_to_ipc(dev_dto), "AddNewDevice");

    // Signal 2: MeshEvent BindAppKeyComp node_ready=true cho Qt app
    EventLogDto evt;
    evt.node_id = node_id_from_unicast(unicast);
    evt.level   = "info";
    evt.event   = "BindAppKeyComp";
    std::ostringstream js;
    js << "{\"node_ready\":true"
       << ",\"node_type\":\"" << node_type << "\""
       << ",\"has_rpr_server\":\"" << (has_rpr_server ? "true" : "false") << "\""
       << ",\"status\":255}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


void GatewayService::on_group_add_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_group_op_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] GroupAdd node=0x" << std::hex << f.addr
              << (ok ? " OK" : " FAILED") << "\n";
    if (!ipc || !p) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = ok ? "info" : "warn";
    evt.event   = "GroupAddComp";
    std::ostringstream js;
    js << "{\"group_addr\":" << (int)p->group_addr
       << ",\"model_id\":"   << (int)p->model_id
       << ",\"company_id\":" << (int)p->company_id
       << ",\"status\":"     << (int)p->status_code << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


void GatewayService::on_group_remove_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_group_op_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] GroupRemove node=0x" << std::hex << f.addr
              << (ok ? " OK" : " FAILED") << "\n";
    if (!ipc || !p) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = ok ? "info" : "warn";
    evt.event   = "GroupRemoveComp";
    std::ostringstream js;
    js << "{\"group_addr\":" << (int)p->group_addr
       << ",\"model_id\":"   << (int)p->model_id
       << ",\"company_id\":" << (int)p->company_id
       << ",\"status\":"     << (int)p->status_code << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


/* ══════════════════════════════════════════════════════════════════════════
   Section 8 — Config Client events
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_comp_data_status(const MeshFrame& f)
{
    if (f.payload.size() < 3) return;  // ít nhất page(1) + data_len(2)

    const uint8_t* raw = f.payload.data();
    uint8_t  page     = raw[0];
    uint16_t data_len = (raw[1] | (raw[2] << 8));
    if (f.payload.size() < (3u + data_len)) return;  // kiểm tra đủ data

    const uint8_t* data = raw + 3;

    std::cout << "[GW] CompData node=0x" << std::hex << f.addr
              << " page=" << (int)page << " len=" << std::dec << data_len << "\n";
              
    if (!ipc) return;
    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "CompDataStatus";
    
    // Đính kèm "data" dạng hex (VD: "0A1B2C...") vào JSON
    evt.payload_json = "{\"page\":" + std::to_string((int)page) + 
                       ", \"data_len\":" + std::to_string(data_len) + 
                       ", \"data\":\"" + GatewayService::bytes_to_hex(data, data_len) + "\"}";
                       
    ipc->send(dto_to_ipc(evt), "MeshEvent");
    on_comp_data_parsed(f.addr, data, data_len);
}

void GatewayService::on_comp_data_parsed(uint16_t unicast,
                                          const uint8_t* data, uint16_t len)
{
    {
        std::lock_guard<std::mutex> lk(prov_mutex_);
        auto it = prov_contexts_.find(unicast);
        if (it == prov_contexts_.end()) return;

        ProvisioningContext& ctx = it->second;
        ctx.bind_queue.clear();
        ctx.bind_idx     = 0;
        ctx.keys_to_add.clear();
        ctx.key_add_idx  = 0;

        if (len < 10) return;
        uint16_t offset   = 10;
        uint8_t  elem_idx = 0;

        while (offset < len) {
            if (offset + 4 > len) break;
            uint8_t nums = data[offset + 2];
            uint8_t numv = data[offset + 3];
            offset += 4;

            // SIG models
            for (int i = 0; i < nums && offset + 2 <= len; i++, offset += 2) {
                uint16_t model_id = data[offset] | (data[offset+1] << 8);
                if (model_id == 0x0000 || model_id == 0x0001) continue;
                if (model_id == 0x0004) {  // ESP_BLE_MESH_MODEL_ID_RPR_SRV
                    ctx.has_rpr_server = true;
                    std::cout << "[GW] Node 0x" << std::hex << unicast 
                            << " has RPR Server\n";
                    continue;
                }
                for (auto& rule : bind_rules_) {
                    if (rule.company_id == 0xFFFF && rule.model_id == model_id) {
                        uint16_t elem_addr = unicast + elem_idx;
                        // FIX Bug 6: 4-tuple, lưu cả app_idx
                        ctx.bind_queue.emplace_back(elem_addr, model_id, (uint16_t)0xFFFF, rule.app_idx);
                        if (rule.node_type == "actuator")
                            ctx.detected_type = "actuator";
                        else if (ctx.detected_type != "actuator" && rule.node_type == "sensor")
                            ctx.detected_type = "sensor";
                        break;
                    }
                }
            }

            // Vendor models
            for (int i = 0; i < numv && offset + 4 <= len; i++, offset += 4) {
                uint16_t company_id = data[offset]   | (data[offset+1] << 8);
                uint16_t model_id   = data[offset+2] | (data[offset+3] << 8);
                for (auto& rule : bind_rules_) {
                    if (rule.company_id == company_id && rule.model_id == model_id) {
                        uint16_t elem_addr = unicast + elem_idx;
                        ctx.bind_queue.emplace_back(elem_addr, model_id, company_id, rule.app_idx);
                        if (rule.node_type == "actuator")
                            ctx.detected_type = "actuator";
                        else if (ctx.detected_type != "actuator")
                            ctx.detected_type = rule.node_type;
                        break;
                    }
                }
            }
            elem_idx++;
        }

        // Collect unique app_idx cần add (giữ thứ tự, không trùng)
        for (auto& [ea, mid, cid, aidx] : ctx.bind_queue) {
            bool found = false;
            for (auto k : ctx.keys_to_add) if (k == aidx) { found = true; break; }
            if (!found) ctx.keys_to_add.push_back(aidx);
        }

        std::cout << "[GW] Node 0x" << std::hex << unicast
                  << " type=" << ctx.detected_type
                  << " bind_queue=" << std::dec << ctx.bind_queue.size()
                  << " keys=" << ctx.keys_to_add.size() << "\n";
    } // mutex released

    advance_provisioning(unicast);
}

// Helper trung tâm: tiến qua các phase (add key → bind model → done)
// Gọi khi KHÔNG giữ prov_mutex_
void GatewayService::advance_provisioning(uint16_t unicast)
{
    std::unique_lock<std::mutex> lk(prov_mutex_);
    auto it = prov_contexts_.find(unicast);
    if (it == prov_contexts_.end()) return;

    ProvisioningContext& ctx = it->second;

    // ── Phase 1: thêm app key vào node ───────────────────────────────
    if (ctx.key_add_idx < ctx.keys_to_add.size()) {
        uint16_t need_idx = ctx.keys_to_add[ctx.key_add_idx];
        // Tìm key config tương ứng
        const AppKeyConfig* cfg = nullptr;
        for (auto& k : app_keys_)
            if (k.app_idx == need_idx) { cfg = &k; break; }

        if (!cfg) {
            std::cerr << "[GW] WARNING: app_idx=" << need_idx 
              << " not in config — skip key, node may not work\n";
            ctx.key_add_idx++;
            lk.unlock();
            advance_provisioning(unicast);  // thử key tiếp
            return;
        }
        // copy để dùng sau khi unlock
        uint16_t net_idx = cfg->net_idx;
        uint16_t app_idx = cfg->app_idx;
        uint8_t  key[16];
        std::memcpy(key, cfg->key, 16);
        lk.unlock();
        sender->add_app_key(unicast, net_idx, app_idx, key);
        return;
    }

    // ── Phase 2: bind model ───────────────────────────────────────────
    if (ctx.bind_idx < ctx.bind_queue.size()) {
        auto [ea, mid, cid, aidx] = ctx.bind_queue[ctx.bind_idx];
        lk.unlock();
        sender->bind_app_key(unicast, ea, aidx, mid, cid);
        return;
    }

    // ── Done ──────────────────────────────────────────────────────────
    std::string detected_type = ctx.detected_type;
    bool      has_rpr_server  = ctx.has_rpr_server;
    prov_contexts_.erase(it);
    lk.unlock();
    on_bind_complete(unicast, detected_type, has_rpr_server);
}

void GatewayService::on_cfg_client_timeout(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_cfg_timeout_t>(f);
    std::cerr << "[GW] Config timeout node=0x" << std::hex << f.addr
              << " opcode=0x" << (p ? (int)p->orig_opcode : 0) << "\n";

    if (p && (p->orig_opcode == BLE_OP_LOW_APP_KEY_ADD || 
          p->orig_opcode == BLE_OP_LOW_APP_BIND)) {
        std::lock_guard<std::mutex> lk(prov_mutex_);
        bool has_ctx = (prov_contexts_.find(f.addr) != prov_contexts_.end());
        lk.~lock_guard(); // release before calling
        
        if (has_ctx) {
            std::cerr << "[GW] Retry prov step node=0x" << std::hex << f.addr << "\n";
            advance_provisioning(f.addr);
        } else {
            std::cerr << "[GW] Timeout but no prov context for 0x" << std::hex << f.addr << "\n";
        }
        return;  
    }
}

/* ══════════════════════════════════════════════════════════════════════════
   Section 9 — Generic Client events
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_onoff_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_onoff_status_t>(f);
    if (!p) return;
    std::cout << "[GW] OnOff node=0x" << std::hex << f.addr
              << " present=" << (int)p->present_onoff
              << " target="  << (int)p->target_onoff << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "OnOffStatus";
    std::ostringstream js;
    js << "{\"present_onoff\":" << (int)p->present_onoff
       << ",\"target_onoff\":"  << (int)p->target_onoff << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_sensor_status(const MeshFrame& f)
{
    // FIX: kiểm tra đủ size trước khi cast
    if (f.payload.size() < sizeof(mesh_evt_sensor_status_t)) {
        std::cerr << "[GW] Sensor status too short: " << f.payload.size() << "\n";
        return;
    }
    auto* p = payload_cast<mesh_evt_sensor_status_t>(f);
    if (!p) return;

    std::cout << "[GW] Sensor node=0x" << std::hex << f.addr
              << " sid=0x"   << p->sensor_id
              << " lux="     << std::dec << p->lux
              << " temp="    << p->temperature
              << " hum="     << (int)p->humidity
              << " motion="  << (int)p->motion
              << " bat="     << (int)p->battery << "\n";

    if (!ipc) return;

    // FIX: fill đầy đủ tất cả fields của SensorDto
    SensorDto dto;
    dto.node_id     = node_id_from_unicast(f.addr);
    dto.sensor_id   = static_cast<int32_t>(p->sensor_id);
    dto.temperature = p->temperature / 10.0;    // int16_t 0.1°C → °C
    dto.humidity    = static_cast<double>(p->humidity);
    dto.lux         = static_cast<double>(p->lux);
    dto.motion      = p->motion;
    dto.battery     = p->battery;
    dto.status      = 0;

    ipc->send(dto_to_ipc(dto), "SensorData");
}

void GatewayService::on_actuator_status(const MeshFrame& f)
{
    // FIX: kiểm tra đủ size trước khi cast
    if (f.payload.size() < sizeof(mesh_evt_actuator_status_t)) {
        std::cerr << "[GW] Actuator status too short: " << f.payload.size() << "\n";
        return;
    }
    auto* p = payload_cast<mesh_evt_actuator_status_t>(f);
    if (!p) return;

    std::cout << "[GW] Actuator node=0x" << std::hex << f.addr
              << " act=0x"    << p->actuator_id
              << " present="  << std::dec << p->present_setpoint
              << " target="   << p->target_setpoint
              << " st=0x"     << std::hex << (int)p->status << "\n";

    if (!ipc) return;

    // FIX: fill đầy đủ present_setpoint và target_setpoint
    ActuatorAckDto ack_dto;
    ack_dto.node_id          = node_id_from_unicast(f.addr);
    ack_dto.actuator_id      = static_cast<int32_t>(p->actuator_id);
    ack_dto.present_setpoint = static_cast<int32_t>(p->present_setpoint);
    ack_dto.target_setpoint  = static_cast<int32_t>(p->target_setpoint);
    ack_dto.success          = (p->status == 0x00);
    ipc->send(dto_to_ipc(ack_dto), "ActuatorAck");
}

void GatewayService::on_generic_client_timeout(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_generic_timeout_t>(f);
    std::cerr << "[GW] Generic timeout node=0x" << std::hex << f.addr
              << " opcode=0x" << (p ? (int)p->orig_opcode : 0) << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "warn";
    evt.event        = "GenericClientTimeout";
    evt.payload_json = "{\"orig_opcode\":" + std::to_string(p ? (int)p->orig_opcode : 0) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


/* ══════════════════════════════════════════════════════════════════════════
   Section 10 — RPR events
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_rpr_scan_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_scan_status_t>(f);
    if (!p) return;
    std::cout << "[GW] RPR ScanStatus srv=0x" << std::hex << f.addr
              << " scanning=" << (int)p->rpr_scanning << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "RprScanStatus";
    std::ostringstream js;
    js << "{\"status\":"           << (int)p->status
       << ",\"rpr_scanning\":"     << (int)p->rpr_scanning
       << ",\"scan_items_limit\":" << (int)p->scan_items_limit
       << ",\"timeout\":"          << (int)p->timeout << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_scan_report(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_scan_report_t>(f);
    if (!p) return;

    std::string uuid_hex = bytes_to_hex(p->uuid, 16);
    std::cout << "[GW] RPR ScanReport srv=0x" << std::hex << f.addr
              << " uuid=" << uuid_hex
              << " rssi=" << std::dec << (int)(int8_t)p->rssi << "\n";

    if (!ipc) return;

    // ── Signal 1: UnprovAdv ───────────────────────────────────────────────
    UnprovAdvDto adv_dto;
    adv_dto.uuid     = uuid_hex;
    adv_dto.mac      = "";   // RPR scan report không có MAC
    adv_dto.rssi     = static_cast<int32_t>((int8_t)p->rssi);
    adv_dto.bearer   = 0;    // Remote provisioning → PB-ADV
    adv_dto.oob_info = static_cast<uint8_t>(p->oob_info & 0xFF);
    ipc->send(dto_to_ipc(adv_dto), "UnprovAdv");

    // ── Signal 2: MeshEvent (log) ─────────────────────────────────────────
    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "RprScanReport";
    std::ostringstream js;
    js << "{\"uuid\":\""   << uuid_hex        << "\""
       << ",\"rssi\":"     << (int)(int8_t)p->rssi
       << ",\"oob_info\":" << (int)p->oob_info << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_link_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_link_status_t>(f);
    if (!p) return;
    std::cout << "[GW] RPR LinkStatus srv=0x" << std::hex << f.addr
              << " state=" << (int)p->rpr_state << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "RprLinkStatus";
    std::ostringstream js;
    js << "{\"status\":" << (int)p->status
       << ",\"rpr_state\":" << (int)p->rpr_state << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_link_open_evt(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_simple_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] RPR LinkOpen srv=0x" << std::hex << f.addr
              << (ok ? " OK" : " FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = ok ? "info" : "warn";
    evt.event        = "RprLinkOpen";
    evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0xFF) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_link_close_evt(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_simple_t>(f);
    std::cout << "[GW] RPR LinkClose srv=0x" << std::hex << f.addr << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "RprLinkClose";
    evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_link_report(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_link_status_t>(f);
    if (!p) return;
    std::cout << "[GW] RPR LinkReport srv=0x" << std::hex << f.addr
              << " state=" << (int)p->rpr_state
              << " reason=" << (int)p->reason << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "RprLinkReport";
    std::ostringstream js;
    js << "{\"status\":"    << (int)p->status
       << ",\"rpr_state\":" << (int)p->rpr_state
       << ",\"reason\":"    << (int)p->reason << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_start_prov_comp(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_simple_t>(f);
    bool  ok = p && p->status_code == 0;
    std::cout << "[GW] RPR StartProv srv=0x" << std::hex << f.addr
              << (ok ? " OK" : " FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = ok ? "info" : "warn";
    evt.event        = "RprStartProvComp";
    evt.payload_json = "{\"status\":" + std::to_string(p ? (int)p->status_code : 0xFF) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_rpr_prov_complete(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_rpr_prov_complete_t>(f);
    if (!p) return;

    std::string uuid_hex = bytes_to_hex(p->uuid, 16);

    {
        std::lock_guard<std::mutex> lk(known_uuids_mutex);
        known_uuids.insert(uuid_hex);
    }

    if (ipc) {
        DeviceProvisionedDto dev_dto;
        dev_dto.node_id  = node_id_from_unicast(p->unicast);
        dev_dto.uuid     = uuid_hex;
        dev_dto.net_idx  = static_cast<int32_t>(p->net_idx);
        dev_dto.elem_num = static_cast<int32_t>(p->elem_num);
        dev_dto.unicast  = static_cast<int32_t>(p->unicast);
        ipc->send(dto_to_ipc(dev_dto), "AddNewDevice");

        EventLogDto evt;
        evt.node_id = dev_dto.node_id;
        evt.level   = "info";
        evt.event   = "RprProvComplete";
        std::ostringstream js;
        js << "{\"unicast\":"   << (int)p->unicast
           << ",\"elem_num\":"  << (int)p->elem_num
           << ",\"net_idx\":"   << (int)p->net_idx
           << ",\"rpr_srv\":"   << (int)f.addr       // thêm info RPR server
           << ",\"uuid\":\""    << uuid_hex << "\"}";
        evt.payload_json = js.str();
        ipc->send(dto_to_ipc(evt), "MeshEvent");
    }

    // FIX: khởi động pipeline CompData/bind cho node MỚI
    // Dùng p->unicast (node mới), KHÔNG phải f.addr (RPR server)
    start_comp_data_pipeline(p->unicast, uuid_hex);
}

/* ══════════════════════════════════════════════════════════════════════════
   Section 11 — Group / Heartbeat events
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_model_subscribe_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_group_op_t>(f);
    if (!p) return;
    std::cout << "[GW] ModelSubStatus node=0x" << std::hex << f.addr
              << " group=0x" << p->group_addr
              << (p->status_code == 0 ? " OK" : " FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "ModelSubscribeStatus";
    std::ostringstream js;
    js << "{\"group_addr\":"  << (int)p->group_addr
       << ",\"model_id\":"    << (int)p->model_id
       << ",\"company_id\":"  << (int)p->company_id
       << ",\"status\":"      << (int)p->status_code << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_model_unsubscribe_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_group_op_t>(f);
    if (!p) return;
    std::cout << "[GW] ModelUnsubStatus node=0x" << std::hex << f.addr
              << " group=0x" << p->group_addr << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "ModelUnsubscribeStatus";
    std::ostringstream js;
    js << "{\"group_addr\":"  << (int)p->group_addr
       << ",\"model_id\":"    << (int)p->model_id
       << ",\"company_id\":"  << (int)p->company_id
       << ",\"status\":"      << (int)p->status_code << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_model_publish_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_group_op_t>(f);
    if (!p) return;
    std::cout << "[GW] ModelPubStatus node=0x" << std::hex << f.addr
              << " pub=0x" << p->group_addr
              << (p->status_code == 0 ? " OK" : " FAILED") << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "ModelPublishStatus";
    std::ostringstream js;
    js << "{\"pub_addr\":"    << (int)p->group_addr
       << ",\"model_id\":"    << (int)p->model_id
       << ",\"company_id\":"  << (int)p->company_id
       << ",\"status\":"      << (int)p->status_code << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_heartbeat(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_heartbeat_t>(f);
    if (!p) return;

    std::cout << "[GW] Heartbeat src=0x" << std::hex << f.addr
              << " hops=" << std::dec << (int)p->hops
              << " ttl="  << (int)p->init_ttl
              << " feat=0x" << std::hex << (int)p->features << "\n";

    if (!ipc) return;

    // ── Signal 1: DeviceStatus (online + features) ────────────────────────
    DeviceStatusDto status_dto;
    status_dto.node_id   = node_id_from_unicast(f.addr);
    status_dto.is_online = true;
    status_dto.features  = static_cast<int32_t>(p->features);
    ipc->send(dto_to_ipc(status_dto), "DeviceStatus");

    // ── Signal 2: MeshEvent (log) ─────────────────────────────────────────
    EventLogDto evt;
    evt.node_id = status_dto.node_id;
    evt.level   = "info";
    evt.event   = "Heartbeat";
    std::ostringstream js;
    js << "{\"init_ttl\":" << (int)p->init_ttl
       << ",\"hops\":"     << (int)p->hops
       << ",\"features\":" << (int)p->features << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_heartbeat_pub_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_heartbeat_pub_status_t>(f);
    if (!p) return;
    std::cout << "[GW] HBPubStatus node=0x" << std::hex << f.addr
              << " dst=0x" << p->dst << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "HeartbeatPubStatus";
    std::ostringstream js;
    js << "{\"status\":" << (int)p->status
       << ",\"dst\":"    << (int)p->dst
       << ",\"period\":" << (int)p->period
       << ",\"ttl\":"    << (int)p->ttl << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_heartbeat_sub_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_heartbeat_sub_status_t>(f);
    if (!p) return;
    std::cout << "[GW] HBSubStatus node=0x" << std::hex << f.addr
              << " dst=0x" << p->dst << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "HeartbeatSubStatus";
    std::ostringstream js;
    js << "{\"status\":" << (int)p->status
       << ",\"dst\":"    << (int)p->dst
       << ",\"period\":" << (int)p->period << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


/* ══════════════════════════════════════════════════════════════════════════
   Section 12 — Network / Feature State events
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::on_relay_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_relay_status_t>(f);
    if (!p) return;
    std::cout << "[GW] RelayStatus node=0x" << std::hex << f.addr
              << " state=" << std::dec << (int)p->relay_state << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id = node_id_from_unicast(f.addr);
    evt.level   = "info";
    evt.event   = "RelayStatus";
    std::ostringstream js;
    js << "{\"relay_state\":"             << (int)p->relay_state
        << ",\"retransmit_count\":"         << (int)p->retransmit_count
        << ",\"retransmit_interval_ms\":"   << (int)p->retransmit_interval_ms
        << "}";
    evt.payload_json = js.str();
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_proxy_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_feature_simple_t>(f);
    if (!p) return;
    std::cout << "[GW] ProxyStatus node=0x" << std::hex << f.addr
              << " state=" << std::dec << (int)p->state << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "ProxyStatus";
    evt.payload_json = "{\"state\":" + std::to_string((int)p->state) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_friend_status(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_feature_simple_t>(f);
    if (!p) return;
    std::cout << "[GW] FriendStatus node=0x" << std::hex << f.addr
              << " state=" << std::dec << (int)p->state << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "FriendStatus";
    evt.payload_json = "{\"state\":" + std::to_string((int)p->state) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_friendship_established(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_friendship_t>(f);
    std::cout << "[GW] FriendshipEstablished node=0x" << std::hex << f.addr
              << " friend=0x" << (p ? p->friend_unicast : 0) << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "FriendshipEstablished";
    evt.payload_json = "{\"friend_unicast\":" + std::to_string(p ? (int)p->friend_unicast : 0) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}

void GatewayService::on_friendship_terminated(const MeshFrame& f)
{
    auto* p = payload_cast<mesh_evt_friendship_t>(f);
    std::cout << "[GW] FriendshipTerminated node=0x" << std::hex << f.addr
              << " friend=0x" << (p ? p->friend_unicast : 0) << "\n";
    if (!ipc) return;

    EventLogDto evt;
    evt.node_id      = node_id_from_unicast(f.addr);
    evt.level        = "info";
    evt.event        = "FriendshipTerminated";
    evt.payload_json = "{\"friend_unicast\":" + std::to_string(p ? (int)p->friend_unicast : 0) + "}";
    ipc->send(dto_to_ipc(evt), "MeshEvent");
}


/* ══════════════════════════════════════════════════════════════════════════
   Public API — Facade (delegate sang MeshCommandSender)
   ══════════════════════════════════════════════════════════════════════════ */

void GatewayService::enable_provisioning()  { sender->prov_enable();  }
void GatewayService::disable_provisioning() { sender->prov_disable(); }

void GatewayService::set_dev_uuid_match(const std::string& hex) {
    uint8_t m[8]{};
    hex_to_bytes(hex, m, std::min(hex.size() / 2, (size_t)8));
    sender->set_dev_uuid_match(m);
}

void GatewayService::add_unprovisioned_device(const std::string& hex, uint8_t bearer) {
    uint8_t uuid[16]{};
    hex_to_bytes(hex, uuid, 16);
    sender->add_unprov_dev(uuid, bearer);
}

void GatewayService::set_node_name(uint16_t u, const std::string& n) { sender->set_node_name(u, n); }
void GatewayService::delete_node  (uint16_t u)                       { sender->delete_node(u); }

void GatewayService::group_add(uint16_t u, uint16_t el, uint16_t gr, uint16_t m, uint16_t c) {
    sender->group_add(u, el, gr, m, c);
}
void GatewayService::group_delete(uint16_t u, uint16_t el, uint16_t gr, uint16_t m, uint16_t c) {
    sender->group_delete(u, el, gr, m, c);
}
void GatewayService::model_pub_set(uint16_t u, uint16_t el, uint16_t pa,
                                    uint16_t m, uint16_t c, uint8_t ttl, uint8_t p) {
    sender->model_pub_set(u, el, pa, m, c, ttl, p);
}

void GatewayService::on_off_get      (uint16_t u)                     { sender->on_off_get(u); }
void GatewayService::on_off_set      (uint16_t u, bool on, uint8_t t) { sender->on_off_set(u, on, t); }
void GatewayService::on_off_set_unack(uint16_t u, bool on, uint8_t t) { sender->on_off_set_unack(u, on, t); }
void GatewayService::sensor_get      (uint16_t u, uint16_t sid)       { sender->sensor_get(u, sid); }
void GatewayService::actuator_get    (uint16_t u)                     { sender->actuator_get(u); }
void GatewayService::actuator_set      (uint16_t u, uint16_t ai, uint16_t sp, uint8_t st) { sender->actuator_set(u, ai, sp, st); }
void GatewayService::actuator_set_unack(uint16_t u, uint16_t ai, uint16_t sp, uint8_t st) { sender->actuator_set_unack(u, ai, sp, st); }

void GatewayService::relay_set    (uint16_t u, uint8_t s, uint8_t rc, uint8_t ri) { sender->relay_set(u, s, rc, ri); }
void GatewayService::relay_get    (uint16_t u)             { sender->relay_get(u); }
void GatewayService::proxy_set    (uint16_t u, uint8_t s)  { sender->proxy_set(u, s); }
void GatewayService::proxy_get    (uint16_t u)             { sender->proxy_get(u); }
void GatewayService::friend_set   (uint16_t u, uint8_t s)  { sender->friend_set(u, s); }
void GatewayService::friend_get   (uint16_t u)             { sender->friend_get(u); }
void GatewayService::heartbeat_sub(uint16_t addr, uint16_t src_addr, uint16_t group_addr, uint16_t period = 0xFF) 
{
    sender->heartbeat_sub(addr, src_addr, group_addr, period);
}
                   
void GatewayService::heartbeat_pub(uint16_t u, uint16_t d, uint8_t p, uint8_t t) { sender->heartbeat_pub(u, d, p, t); }

void GatewayService::rpr_scan_get  (uint16_t s)                                               { sender->rpr_scan_get(s); }
void GatewayService::rpr_scan_stop (uint16_t s)                                               { sender->rpr_scan_stop(s); }
void GatewayService::rpr_link_get  (uint16_t s)                                               { sender->rpr_link_get(s); }
void GatewayService::rpr_start_prov(uint16_t s)                                               { sender->rpr_start_prov(s); }
void GatewayService::rpr_scan_start(uint16_t s, uint8_t lim, uint8_t to, bool f, const uint8_t* u) { sender->rpr_scan_start(s, lim, to, f, u); }
void GatewayService::rpr_link_open (uint16_t s, const uint8_t u[16], bool te, uint8_t to)     { sender->rpr_link_open(s, u, te, to); }
void GatewayService::rpr_link_close(uint16_t s, uint8_t r)                                    { sender->rpr_link_close(s, r); }


/* ══════════════════════════════════════════════════════════════════════════
   Static helpers
   ══════════════════════════════════════════════════════════════════════════ */

/*
 * node_id_from_unicast:
 *   Format: "node_XXXX" với XXXX là unicast addr (hex, 4 chữ số)
 *   Ví dụ: 0x0005 → "node_0005"
 *          0x0010 → "node_0010"
 *
 * Database daemon sẽ resolve tên thiết bị từ DB khi cần hiển thị.
 * GatewayService chỉ cần key nhất quán để lookup.
 */
std::string GatewayService::node_id_from_unicast(uint16_t unicast)
{
    std::ostringstream ss;
    ss << "node_" << std::hex << std::setw(4) << std::setfill('0') << unicast;
    return ss.str();
}

/*
 * unicast_from_node_id:
 *   Parse "node_XXXX" → unicast addr
 *   Trả về 0 nếu format không đúng
 */
uint16_t GatewayService::unicast_from_node_id(const std::string& node_id)
{
    if (node_id.rfind("node_", 0) == 0 && node_id.size() > 5) {
        try {
            return static_cast<uint16_t>(
                std::stoul(node_id.substr(5), nullptr, 16));
        } catch (...) {}
    }
    return 0;
}

bool GatewayService::hex_to_bytes(const std::string& hex, uint8_t* out, size_t len)
{
    if (hex.size() < len * 2) return false;
    for (size_t i = 0; i < len; i++)
        out[i] = static_cast<uint8_t>(std::stoul(hex.substr(i * 2, 2), nullptr, 16));
    return true;
}

std::string GatewayService::bytes_to_hex(const uint8_t* data, size_t len)
{
    std::ostringstream ss;
    ss << std::hex << std::setfill('0');
    for (size_t i = 0; i < len; i++)
        ss << std::setw(2) << static_cast<int>(data[i]);
    return ss.str();
}



/// Day la module dbus cua cac app database, daemon app dung de giao tiep voiw dbus cua qt o tren
#ifndef IPC_DTO_H
#define IPC_DTO_H

/**
 * @file ipc_dto.h  (GLib side — GatewayApp / DatabaseApp / MqttApp)
 *
 * Tất cả DTO đều serialize/deserialize sang JSON string.
 * Wire format: IpcMessage.payload = g_variant_new_string(json_cstr)
 *
 * Không còn dùng GVariant typed tuple (ssiii, sidddyyi, ...).
 * Mỗi DTO có:
 *   std::string to_json() const
 *   static TDto  from_json(const std::string& json)
 *   IpcMessage   to_ipc()  const   [= wrap to_json() vào GVariant string]
 *   static TDto  from_ipc(const IpcMessage& msg)
 */

#include "ipc_message.h"
#include "ipc_json_utils.h"

#include <string>
#include <cstdint>
#include <sstream>
#include <stdexcept>

// ─────────────────────────────────────────────────────────────────────────────
//  Macro helper
// ─────────────────────────────────────────────────────────────────────────────

#define IPC_FROM_JSON(Type) \
    static Type from_ipc(const IpcMessage& msg) { \
        return from_json(jutil::from_gvariant(msg.payload)); \
    } \
    IpcMessage to_ipc() const { \
        IpcMessage m; \
        m.payload = jutil::to_gvariant(to_json()); \
        return m; \
    }

// ─────────────────────────────────────────────────────────────────────────────
//  DeviceProvisionedDto
//  Emitted by: on_prov_complete, on_rpr_prov_complete
//  Signal name: "AddNewDevice"
// ─────────────────────────────────────────────────────────────────────────────
struct DeviceProvisionedDto {
    std::string node_id;  // "node_XXXX"
    std::string uuid;     // hex 32 chars
    int32_t     net_idx  = 0;
    int32_t     elem_num = 1;
    int32_t     unicast  = 0;
    std::string node_type;

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""  << jutil::esc(node_id) << "\","
          << "\"uuid\":\""     << jutil::esc(uuid)    << "\","
          << "\"net_idx\":"    << net_idx              << ","
          << "\"elem_num\":"   << elem_num             << ","
          << "\"unicast\":"    << unicast              << ","
          << "\"node_type\":"  << node_type
          << "}";
        return o.str();
    }

    static DeviceProvisionedDto from_json(const std::string& j) {
        DeviceProvisionedDto d;
        d.node_id  = jutil::get_str  (j, "node_id");
        d.uuid     = jutil::get_str  (j, "uuid");
        d.net_idx  = jutil::get_int  (j, "net_idx");
        d.elem_num = jutil::get_int  (j, "elem_num", 1);
        d.unicast  = jutil::get_int  (j, "unicast");
        d.node_type= jutil::get_str  (j, "node_type");
        return d;
    }

    IPC_FROM_JSON(DeviceProvisionedDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  SensorDto
//  Emitted by: on_sensor_status (EVT_SENSOR_STATUS = 0xB1)
//  Signal name: "SensorData"
// ─────────────────────────────────────────────────────────────────────────────
struct SensorDto {
    std::string node_id;
    int32_t     sensor_id   = 0;
    double      temperature = 0.0;  // °C (float)
    double      humidity    = 0.0;  // %
    double      lux         = 0.0;
    int32_t     motion      = 0;    // 0/1
    int32_t     battery     = 0;    // %
    int32_t     status      = 0;

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""     << jutil::esc(node_id) << "\","
          << "\"sensor_id\":"     << sensor_id            << ","
          << "\"temperature\":"   << temperature          << ","
          << "\"humidity\":"      << humidity             << ","
          << "\"lux\":"           << lux                  << ","
          << "\"motion\":"        << motion               << ","
          << "\"battery\":"       << battery              << ","
          << "\"status\":"        << status
          << "}";
        return o.str();
    }

    static SensorDto from_json(const std::string& j) {
        SensorDto s;
        s.node_id     = jutil::get_str   (j, "node_id");
        s.sensor_id   = jutil::get_int   (j, "sensor_id");
        s.temperature = jutil::get_double(j, "temperature");
        s.humidity    = jutil::get_double(j, "humidity");
        s.lux         = jutil::get_double(j, "lux");
        s.motion      = jutil::get_int   (j, "motion");
        s.battery     = jutil::get_int   (j, "battery");
        s.status      = jutil::get_int   (j, "status");
        return s;
    }

    IPC_FROM_JSON(SensorDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  ActuatorAckDto
//  Emitted by: on_actuator_status (EVT_ACTUATOR_STATUS = 0xB2)
//  Signal name: "ActuatorAck"
// ─────────────────────────────────────────────────────────────────────────────
struct ActuatorAckDto {
    std::string node_id;
    int32_t     actuator_id      = 0;
    int32_t     present_setpoint = 0;
    int32_t     target_setpoint  = 0;
    bool        success          = false;

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""          << jutil::esc(node_id) << "\","
          << "\"actuator_id\":"         << actuator_id          << ","
          << "\"present_setpoint\":"    << present_setpoint     << ","
          << "\"target_setpoint\":"     << target_setpoint      << ","
          << "\"success\":"             << (success ? "true" : "false")
          << "}";
        return o.str();
    }

    static ActuatorAckDto from_json(const std::string& j) {
        ActuatorAckDto a;
        a.node_id          = jutil::get_str (j, "node_id");
        a.actuator_id      = jutil::get_int (j, "actuator_id");
        a.present_setpoint = jutil::get_int (j, "present_setpoint");
        a.target_setpoint  = jutil::get_int (j, "target_setpoint");
        a.success          = jutil::get_bool(j, "success");
        return a;
    }

    IPC_FROM_JSON(ActuatorAckDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  DeviceStatusDto
//  Emitted by: on_node_reset, on_heartbeat
//  Signal name: "DeviceStatus"
// ─────────────────────────────────────────────────────────────────────────────
struct DeviceStatusDto {
    std::string node_id;
    bool        is_online = false;
    int32_t     features  = 0;  // bitfield from EVT_HEARTBEAT

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""  << jutil::esc(node_id)            << "\","
          << "\"is_online\":"  << (is_online ? "true" : "false")  << ","
          << "\"features\":"   << features
          << "}";
        return o.str();
    }

    static DeviceStatusDto from_json(const std::string& j) {
        DeviceStatusDto d;
        d.node_id   = jutil::get_str (j, "node_id");
        d.is_online = jutil::get_bool(j, "is_online");
        d.features  = jutil::get_int (j, "features");
        return d;
    }

    IPC_FROM_JSON(DeviceStatusDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  EventLogDto
//  Used for: MeshEvent signal (all event log messages)
//  Signal name: "MeshEvent", "EventLog"
// ─────────────────────────────────────────────────────────────────────────────
struct EventLogDto {
    std::string node_id;
    std::string level;        // "info" | "warn" | "error"
    std::string event;        // "ProvComplete", "HealthFault", ...
    std::string payload_json; // nested JSON string (not double-encoded)

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""      << jutil::esc(node_id) << "\","
          << "\"level\":\""        << jutil::esc(level)   << "\","
          << "\"event\":\""        << jutil::esc(event)   << "\","
          << "\"payload\":"        << (payload_json.empty() ? "{}" : payload_json)
          << "}";
        return o.str();
    }

    static EventLogDto from_json(const std::string& j) {
        EventLogDto e;
        e.node_id = jutil::get_str(j, "node_id");
        e.level   = jutil::get_str(j, "level", "info");
        e.event   = jutil::get_str(j, "event");
        // payload là nested object — lấy raw string
        const std::string pat = "\"payload\":";
        auto pos = j.find(pat);
        if (pos != std::string::npos) {
            pos += pat.size();
            while (pos < j.size() && (j[pos] == ' ' || j[pos] == '\t')) ++pos;
            if (pos < j.size() && j[pos] == '{') {
                // đọc đến closing brace (shallow — đủ cho payload đơn giản)
                int depth = 0;
                auto start = pos;
                for (; pos < j.size(); ++pos) {
                    if (j[pos] == '{') ++depth;
                    else if (j[pos] == '}') { --depth; if (depth == 0) { ++pos; break; } }
                }
                e.payload_json = j.substr(start, pos - start);
            } else if (pos < j.size() && j[pos] == '"') {
                // payload encoded as string
                e.payload_json = jutil::get_str(j, "payload");
            } else {
                e.payload_json = "{}";
            }
        } else {
            e.payload_json = "{}";
        }
        return e;
    }

    IPC_FROM_JSON(EventLogDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  UnprovAdvDto
//  Emitted by: on_recv_unprov_adv_pkt, on_rpr_scan_report
//  Signal name: "UnprovAdv"
// ─────────────────────────────────────────────────────────────────────────────
struct UnprovAdvDto {
    std::string uuid;     // hex 32 chars
    std::string mac;      // hex 12 chars (bd_addr), empty if RPR
    int32_t     rssi     = 0;
    int32_t     bearer   = 0;   // 0=PB-ADV, 1=PB-GATT
    int32_t     oob_info = 0;

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"uuid\":\""     << jutil::esc(uuid) << "\","
          << "\"mac\":\""      << jutil::esc(mac)  << "\","
          << "\"rssi\":"       << rssi              << ","
          << "\"bearer\":"     << bearer            << ","
          << "\"oob_info\":"   << oob_info
          << "}";
        return o.str();
    }

    static UnprovAdvDto from_json(const std::string& j) {
        UnprovAdvDto d;
        d.uuid     = jutil::get_str(j, "uuid");
        d.mac      = jutil::get_str(j, "mac");
        d.rssi     = jutil::get_int(j, "rssi");
        d.bearer   = jutil::get_int(j, "bearer");
        d.oob_info = jutil::get_int(j, "oob_info");
        return d;
    }

    IPC_FROM_JSON(UnprovAdvDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  ActuatorDto
//  Used for: SendActuatorCmd method call (UI → GatewayService)
// ─────────────────────────────────────────────────────────────────────────────
struct ActuatorDto {
    std::string node_id;
    double      setpoint    = 0.0;
    bool        onoff       = false;
    int32_t     actuator_id = 0;
    std::string status      = "pending";

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""     << jutil::esc(node_id)           << "\","
          << "\"setpoint\":"      << setpoint                       << ","
          << "\"onoff\":"         << (onoff ? "true" : "false")    << ","
          << "\"actuator_id\":"   << actuator_id                    << ","
          << "\"status\":\""      << jutil::esc(status)            << "\""
          << "}";
        return o.str();
    }

    static ActuatorDto from_json(const std::string& j) {
        ActuatorDto a;
        a.node_id     = jutil::get_str (j, "node_id");
        a.setpoint    = jutil::get_double(j, "setpoint");
        a.onoff       = jutil::get_bool(j, "onoff");
        a.actuator_id = jutil::get_int (j, "actuator_id");
        a.status      = jutil::get_str (j, "status", "pending");
        return a;
    }

    IPC_FROM_JSON(ActuatorDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  UuidWhitelistDto
//  Used for: SetUuidMatch method, WhitelistAdd/Remove
// ─────────────────────────────────────────────────────────────────────────────
struct UuidWhitelistDto {
    std::string uuid;
    int32_t     bearer = 0;

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"uuid\":\""   << jutil::esc(uuid) << "\","
          << "\"bearer\":"   << bearer
          << "}";
        return o.str();
    }

    static UuidWhitelistDto from_json(const std::string& j) {
        UuidWhitelistDto d;
        d.uuid   = jutil::get_str(j, "uuid");
        if (d.uuid.empty()) d.uuid = jutil::get_str(j, "uuid_match_hex"); // legacy key
        d.bearer = jutil::get_int(j, "bearer");
        return d;
    }

    IPC_FROM_JSON(UuidWhitelistDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  NodeQueryDto
//  Used for: DB query methods (node_id + optional limit/page)
// ─────────────────────────────────────────────────────────────────────────────
struct NodeQueryDto {
    std::string node_id;
    int32_t     limit = 50;

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\"" << jutil::esc(node_id) << "\","
          << "\"limit\":"     << limit
          << "}";
        return o.str();
    }

    static NodeQueryDto from_json(const std::string& j) {
        NodeQueryDto q;
        q.node_id = jutil::get_str(j, "node_id");
        q.limit   = jutil::get_int(j, "limit", 50);
        return q;
    }

    IPC_FROM_JSON(NodeQueryDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  ConfigCmdDto  (NEW — thay thế makeConfigMsg trong Controller Qt)
//  Used for: SetNodeName, RelaySet, ProxySet, FriendSet, GroupAdd,
//            GroupDelete, PublishToGroup, HeartbeatPub, HeartbeatSub,
//            FotaUpdate, GetCompData, ...
//
//  Format: { "node_id": "...", "event": "...", "payload": { ... } }
// ─────────────────────────────────────────────────────────────────────────────
struct ConfigCmdDto {
    std::string node_id;
    std::string event;
    std::string payload_json = "{}"; // nested JSON object

    std::string to_json() const {
        std::ostringstream o;
        o << "{"
          << "\"node_id\":\""  << jutil::esc(node_id) << "\","
          << "\"event\":\""    << jutil::esc(event)   << "\","
          << "\"payload\":"    << (payload_json.empty() ? "{}" : payload_json)
          << "}";
        return o.str();
    }

    static ConfigCmdDto from_json(const std::string& j) {
        ConfigCmdDto c;
        c.node_id = jutil::get_str(j, "node_id");
        c.event   = jutil::get_str(j, "event");
        // extract nested payload object
        const std::string pat = "\"payload\":";
        auto pos = j.find(pat);
        if (pos != std::string::npos) {
            pos += pat.size();
            while (pos < j.size() && (j[pos] == ' ' || j[pos] == '\t')) ++pos;
            if (pos < j.size() && j[pos] == '{') {
                int depth = 0;
                auto start = pos;
                for (; pos < j.size(); ++pos) {
                    if (j[pos] == '{') ++depth;
                    else if (j[pos] == '}') { --depth; if (depth == 0) { ++pos; break; } }
                }
                c.payload_json = j.substr(start, pos - start);
            } else if (pos < j.size() && j[pos] == '"') {
                c.payload_json = jutil::get_str(j, "payload");
            }
        }
        return c;
    }

    IPC_FROM_JSON(ConfigCmdDto)
};


// ─────────────────────────────────────────────────────────────────────────────
//  ProvisionStartDto
// ─────────────────────────────────────────────────────────────────────────────
struct ProvisionStartDto {
    std::string uuid_hex;
    int32_t     bearer = 0;

    std::string to_json() const {
        return std::string("{")
            + "\"uuid_hex\":\"" + jutil::esc(uuid_hex) + "\","
            + "\"bearer\":" + std::to_string(bearer)
            + "}";
    }
    static ProvisionStartDto from_json(const std::string& j) {
        ProvisionStartDto d;
        d.uuid_hex = jutil::get_str(j, "uuid_hex");
        d.bearer   = jutil::get_int(j, "bearer", 0);
        return d;
    }
    IPC_FROM_JSON(ProvisionStartDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  DeleteNodeDto
// ─────────────────────────────────────────────────────────────────────────────
struct DeleteNodeDto {
    std::string node_id;
    std::string uuid_hex;

    std::string to_json() const {
        return std::string("{")
            + "\"node_id\":\"" + jutil::esc(node_id) + "\","
            + "\"uuid_hex\":\"" + jutil::esc(uuid_hex) + "\""
            + "}";
    }
    static DeleteNodeDto from_json(const std::string& j) {
        DeleteNodeDto d;
        d.node_id  = jutil::get_str(j, "node_id");
        d.uuid_hex = jutil::get_str(j, "uuid_hex");
        return d;
    }
    IPC_FROM_JSON(DeleteNodeDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  SubscribeGroupDto (group add)
// ─────────────────────────────────────────────────────────────────────────────
struct SubscribeGroupDto {
    std::string node_id;
    uint16_t    element_addr = 0;
    uint16_t    group_addr   = 0;
    uint16_t    model_id     = 0;
    uint16_t    company_id   = 0xFFFF;

    std::string to_json() const {
        char buf[256];
        snprintf(buf, sizeof(buf),
            "{\"node_id\":\"%s\",\"element_addr\":%u,\"group_addr\":%u,\"model_id\":%u,\"company_id\":%u}",
            jutil::esc(node_id).c_str(), element_addr, group_addr, model_id, company_id);
        return buf;
    }
    static SubscribeGroupDto from_json(const std::string& j) {
        SubscribeGroupDto d;
        d.node_id      = jutil::get_str(j, "node_id");
        d.element_addr = static_cast<uint16_t>(jutil::get_int(j, "element_addr", 0));
        d.group_addr   = static_cast<uint16_t>(jutil::get_int(j, "group_addr", 0));
        d.model_id     = static_cast<uint16_t>(jutil::get_int(j, "model_id", 0));
        d.company_id   = static_cast<uint16_t>(jutil::get_int(j, "company_id", 0xFFFF));
        return d;
    }
    IPC_FROM_JSON(SubscribeGroupDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  PublishGroupDto
// ─────────────────────────────────────────────────────────────────────────────
struct PublishGroupDto {
    std::string node_id;
    uint16_t    element_addr = 0;
    uint16_t    pub_addr     = 0;
    uint16_t    model_id     = 0;
    uint16_t    company_id   = 0xFFFF;
    uint8_t     ttl          = 7;
    uint8_t     period       = 0;

    std::string to_json() const {
        char buf[256];
        snprintf(buf, sizeof(buf),
            "{\"node_id\":\"%s\",\"element_addr\":%u,\"pub_addr\":%u,\"model_id\":%u,\"company_id\":%u,\"ttl\":%u,\"period\":%u}",
            jutil::esc(node_id).c_str(), element_addr, pub_addr, model_id, company_id, ttl, period);
        return buf;
    }
    static PublishGroupDto from_json(const std::string& j) {
        PublishGroupDto d;
        d.node_id      = jutil::get_str(j, "node_id");
        d.element_addr = static_cast<uint16_t>(jutil::get_int(j, "element_addr", 0));
        d.pub_addr     = static_cast<uint16_t>(jutil::get_int(j, "pub_addr", 0));
        d.model_id     = static_cast<uint16_t>(jutil::get_int(j, "model_id", 0));
        d.company_id   = static_cast<uint16_t>(jutil::get_int(j, "company_id", 0xFFFF));
        d.ttl          = static_cast<uint8_t>(jutil::get_int(j, "ttl", 7));
        d.period       = static_cast<uint8_t>(jutil::get_int(j, "period", 0));
        return d;
    }
    IPC_FROM_JSON(PublishGroupDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  HeartbeatPubDto
// ─────────────────────────────────────────────────────────────────────────────
struct HeartbeatPubDto {
    std::string node_id;
    uint16_t    dst    = 0;
    uint8_t     period = 0;
    uint8_t     ttl    = 7;

    std::string to_json() const {
        char buf[128];
        snprintf(buf, sizeof(buf),
            "{\"node_id\":\"%s\",\"dst\":%u,\"period\":%u,\"ttl\":%u}",
            jutil::esc(node_id).c_str(), dst, period, ttl);
        return buf;
    }
    static HeartbeatPubDto from_json(const std::string& j) {
        HeartbeatPubDto d;
        d.node_id = jutil::get_str(j, "node_id");
        d.dst     = static_cast<uint16_t>(jutil::get_int(j, "dst", 0));
        d.period  = static_cast<uint8_t>(jutil::get_int(j, "period", 0));
        d.ttl     = static_cast<uint8_t>(jutil::get_int(j, "ttl", 7));
        return d;
    }
    IPC_FROM_JSON(HeartbeatPubDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  HeartbeatSubDto
// ─────────────────────────────────────────────────────────────────────────────
struct HeartbeatSubDto {
    std::string node_id;
    uint16_t    group_addr = 0;

    std::string to_json() const {
        char buf[128];
        snprintf(buf, sizeof(buf),
            "{\"node_id\":\"%s\",\"group_addr\":%u}",
            jutil::esc(node_id).c_str(), group_addr);
        return buf;
    }
    static HeartbeatSubDto from_json(const std::string& j) {
        HeartbeatSubDto d;
        d.node_id    = jutil::get_str(j, "node_id");
        d.group_addr = static_cast<uint16_t>(jutil::get_int(j, "group_addr", 0));
        return d;
    }
    IPC_FROM_JSON(HeartbeatSubDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  RprScanStartDto
// ─────────────────────────────────────────────────────────────────────────────
struct RprScanStartDto {
    std::string node_id;   // server node
    std::string uuid;
    uint8_t     limit          = 10;
    uint8_t     timeout        = 10;
    bool        uuid_filter_en = false;

    std::string to_json() const {
        char buf[256];
        snprintf(buf, sizeof(buf),
            "{\"node_id\":\"%s\",\"uuid\":\"%s\",\"limit\":%u,\"timeout\":%u,\"uuid_filter_en\":%s}",
            jutil::esc(node_id).c_str(), jutil::esc(uuid).c_str(), limit, timeout,
            uuid_filter_en ? "true" : "false");
        return buf;
    }
    static RprScanStartDto from_json(const std::string& j) {
        RprScanStartDto d;
        d.node_id        = jutil::get_str(j, "node_id");
        d.uuid           = jutil::get_str(j, "uuid");
        d.limit          = static_cast<uint8_t>(jutil::get_int(j, "limit", 10));
        d.timeout        = static_cast<uint8_t>(jutil::get_int(j, "timeout", 10));
        d.uuid_filter_en = jutil::get_bool(j, "uuid_filter_en", false);
        return d;
    }
    IPC_FROM_JSON(RprScanStartDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  RprLinkOpenDto
// ─────────────────────────────────────────────────────────────────────────────
struct RprLinkOpenDto {
    std::string node_id;   // server node
    std::string uuid;
    bool        timeout_en = false;
    uint8_t     timeout    = 0;

    std::string to_json() const {
        char buf[256];
        snprintf(buf, sizeof(buf),
            "{\"node_id\":\"%s\",\"uuid\":\"%s\",\"timeout_en\":%s,\"timeout\":%u}",
            jutil::esc(node_id).c_str(), jutil::esc(uuid).c_str(),
            timeout_en ? "true" : "false", timeout);
        return buf;
    }
    static RprLinkOpenDto from_json(const std::string& j) {
        RprLinkOpenDto d;
        d.node_id    = jutil::get_str(j, "node_id");
        d.uuid       = jutil::get_str(j, "uuid");
        d.timeout_en = jutil::get_bool(j, "timeout_en", false);
        d.timeout    = static_cast<uint8_t>(jutil::get_int(j, "timeout", 0));
        return d;
    }
    IPC_FROM_JSON(RprLinkOpenDto)
};

// ─────────────────────────────────────────────────────────────────────────────
//  Global helpers: dto_to_ipc / ipc_to_dto
// ─────────────────────────────────────────────────────────────────────────────

template<typename TDto>
inline IpcMessage dto_to_ipc(const TDto& dto) {
    return dto.to_ipc();
}

template<typename TDto>
inline TDto ipc_to_dto(const IpcMessage& msg) {
    return TDto::from_ipc(msg);
}

#endif // IPC_DTO_H

#ifndef _JSON_UTILS_H
#define _JSON_UTILS_H

/**
 * @file ipc_json_utils.h
 * @brief Lightweight JSON helper — không dùng nlohmann/rapidjson, chỉ dùng
 *        std::string + std::ostringstream để tránh thêm dependency.
 *
 * Quy ước:
 *   - Tất cả IPC payload đều là JSON string wrapped trong GVariant string:
 *       GVariant* payload = g_variant_new_string(json_cstr)
 *   - Bên nhận unwrap:
 *       const gchar* s = g_variant_get_string(payload, nullptr);
 *       auto dto = XxxDto::from_json(s);
 */

#include <string>
#include <sstream>
#include <stdexcept>
#include <cstdint>

// ─────────────────────────────────────────────────────────────────────────────
//  Low-level string helpers
// ─────────────────────────────────────────────────────────────────────────────

namespace jutil {

/// Escape special chars cho JSON string value
inline std::string esc(const std::string& s) {
    std::string r;
    r.reserve(s.size() + 4);
    for (unsigned char c : s) {
        switch (c) {
            case '"':  r += "\\\""; break;
            case '\\': r += "\\\\"; break;
            case '\n': r += "\\n";  break;
            case '\r': r += "\\r";  break;
            case '\t': r += "\\t";  break;
            default:
                if (c < 0x20) {
                    // control chars
                    char buf[8];
                    snprintf(buf, sizeof(buf), "\\u%04x", (unsigned)c);
                    r += buf;
                } else {
                    r += (char)c;
                }
                break;
        }
    }
    return r;
}

/// Trích xuất giá trị string từ JSON: "key":"VALUE"
inline std::string get_str(const std::string& json,
                            const std::string& key,
                            const std::string& def = "") {
    const std::string pat = "\"" + key + "\":\"";
    auto pos = json.find(pat);
    if (pos == std::string::npos) return def;
    pos += pat.size();
    std::string val;
    bool escape = false;
    for (; pos < json.size(); ++pos) {
        char c = json[pos];
        if (escape) {
            switch (c) {
                case '"':  val += '"';  break;
                case '\\': val += '\\'; break;
                case 'n':  val += '\n'; break;
                case 'r':  val += '\r'; break;
                case 't':  val += '\t'; break;
                default:   val += c;    break;
            }
            escape = false;
        } else if (c == '\\') {
            escape = true;
        } else if (c == '"') {
            break;
        } else {
            val += c;
        }
    }
    return val;
}

/// Trích xuất giá trị số (int / double / bool) từ JSON: "key":VALUE
inline std::string get_raw(const std::string& json,
                            const std::string& key,
                            const std::string& def = "0") {
    const std::string pat = "\"" + key + "\":";
    auto pos = json.find(pat);
    if (pos == std::string::npos) return def;
    pos += pat.size();
    // bỏ qua khoảng trắng
    while (pos < json.size() && (json[pos] == ' ' || json[pos] == '\t')) ++pos;
    if (pos >= json.size()) return def;
    // đọc đến dấu , hoặc }
    auto end = pos;
    while (end < json.size() && json[end] != ',' && json[end] != '}' &&
           json[end] != '\n' && json[end] != ' ') ++end;
    return json.substr(pos, end - pos);
}

inline double get_double(const std::string& json, const std::string& key,
                          double def = 0.0) {
    auto raw = get_raw(json, key, "");
    if (raw.empty()) return def;
    try { return std::stod(raw); } catch (...) { return def; }
}

inline int64_t get_int64(const std::string& json, const std::string& key,
                          int64_t def = 0) {
    auto raw = get_raw(json, key, "");
    if (raw.empty()) return def;
    try { return std::stoll(raw); } catch (...) { return def; }
}

inline int get_int(const std::string& json, const std::string& key, int def = 0) {
    return static_cast<int>(get_int64(json, key, def));
}

inline bool get_bool(const std::string& json, const std::string& key,
                      bool def = false) {
    auto raw = get_raw(json, key, "");
    if (raw == "true" || raw == "1") return true;
    if (raw == "false" || raw == "0") return false;
    return def;
}

/// Unwrap GVariant string → std::string (trả về "{}" nếu null / sai type)
inline std::string from_gvariant(GVariant* v) {
    if (!v) return "{}";
    // Nếu là variant wrapper → unwrap
    GVariant* inner = v;
    bool need_unref = false;
    if (g_variant_is_of_type(v, G_VARIANT_TYPE_VARIANT)) {
        inner = g_variant_get_variant(v);
        need_unref = true;
    }
    std::string result = "{}";
    if (g_variant_is_of_type(inner, G_VARIANT_TYPE_STRING)) {
        const gchar* s = g_variant_get_string(inner, nullptr);
        if (s) result = s;
    }
    if (need_unref) g_variant_unref(inner);
    return result;
}

/// Wrap JSON string → GVariant string (caller owns ref)
inline GVariant* to_gvariant(const std::string& json) {
    return g_variant_new_string(json.c_str());
}

/// Wrap JSON string → GVariant*(v) wrapper — dùng cho IpcMessage.payload
inline GVariant* to_gvariant_wrapped(const std::string& json) {
    return g_variant_new_variant(g_variant_new_string(json.c_str()));
}

} // namespace jutil

#endif // IPC_JSON_UTILS_H

#ifndef DB_TRANSLATOR_H
#define DB_TRANSLATOR_H

/**
 * @file db_translator.h  (DatabaseApp — GLib side)
 *
 * Tất cả handler đều nhận/trả IpcMessage với payload là JSON string.
 * Không còn GVariant typed tuple.
 *
 * Response format cho method call:
 *   Single object : { "data": { ... } }
 *   Array         : { "data": [ ... ] }
 *   Error         : { "error": "message" }
 *   OK (no data)  : { "ok": true }
 */

#include "ipc.h"
#include "ipc_message.h"
#include "ipc_dto.h"         // JSON-based DTOs (GLib side)
#include "ipc_json_utils.h"
#include "models.h"
#include "repository_device.h"
#include "repository_unprov.h"
#include "repository_sensor.h"
#include "repository_actuator.h"
#include "repository_eventlog.h"
#include "repository_group.h"

#include <functional>
#include <iostream>
#include <sstream>
#include <string>
#include <ctime>
#include <vector>

// ─────────────────────────────────────────────────────────────────────────────
//  Response helpers
// // ─────────────────────────────────────────────────────────────────────────────
// namespace dbt_resp {

namespace dbt_resp {

    inline IpcMessage ok(const std::string& data_json) {
        IpcMessage m;
        // Dùng wrapped để đảm bảo D-Bus method reply nhận được đúng
        m.payload = jutil::to_gvariant_wrapped("{\"data\":" + data_json + "}");
        return m;
    }

    inline IpcMessage ok_flag() {
        IpcMessage m;
        m.payload = jutil::to_gvariant_wrapped("{\"ok\":true}");
        return m;
    }

    inline IpcMessage err(const std::string& msg) {
        IpcMessage m;
        m.payload = jutil::to_gvariant_wrapped(
            "{\"error\":\"" + jutil::esc(msg) + "\"}");
        return m;
    }

} // namespace dbt_resp

// inline IpcMessage ok(const std::string& data_json) {
//     IpcMessage m;
//     m.payload = jutil::to_gvariant("{\"data\":" + data_json + "}");
//     return m;
// }

// inline IpcMessage ok_flag() {
//     IpcMessage m;
//     m.payload = jutil::to_gvariant("{\"ok\":true}");
//     return m;
// }

// inline IpcMessage err(const std::string& msg) {
//     IpcMessage m;
//     m.payload = jutil::to_gvariant(
//         "{\"error\":\"" + jutil::esc(msg) + "\"}");
//     return m;
// }

// } // namespace dbt_resp

// ─────────────────────────────────────────────────────────────────────────────
//  JSON serializers for DB model structs
// ─────────────────────────────────────────────────────────────────────────────
namespace to_json {

inline std::string device(const Device& d) {
    std::ostringstream o;
    o << "{"
      << "\"node_id\":\""          << jutil::esc(d.node_id)          << "\","
      << "\"gateway_id\":\""       << jutil::esc(d.gateway_id)        << "\","
      << "\"net_idx\":"            << d.net_idx                       << ","
      << "\"elem_num\":"           << d.elem_num                      << ","
      << "\"name\":\""             << jutil::esc(d.name)              << "\","
      << "\"type\":\""             << jutil::esc(d.type)              << "\","
      << "\"status\":\""           << jutil::esc(d.status)            << "\","
      << "\"unicast\":"            << d.unicast                       << ","
      << "\"uuid\":\""             << jutil::esc(d.uuid)              << "\","
      << "\"mac\":\""              << jutil::esc(d.mac)               << "\","
      << "\"relay\":"              << d.relay                         << ","
      << "\"proxy\":"              << d.proxy                         << ","
      << "\"friend\":"             << d.friend_                       << ","
      << "\"lpn\":"                << d.lpn                           << ","
      << "\"ttl\":"                << d.ttl                           << ","
      << "\"features\":"           << d.features                      << ","
      << "\"firmware_version\":\"" << jutil::esc(d.firmware_version)  << "\","
      << "\"last_seen\":"          << d.last_seen
      << "}";
    return o.str();
}

inline std::string unprov(const UnprovisionedDevice& u) {
    std::ostringstream o;
    o << "{"
      << "\"uuid\":\""    << jutil::esc(u.uuid)   << "\","
      << "\"mac\":\""     << jutil::esc(u.mac)    << "\","
      << "\"rssi\":"      << u.rssi               << ","
      << "\"oob_info\":"  << u.oob_info           << ","
      << "\"bearer\":"    << u.bearer             << ","
      << "\"status\":\""  << jutil::esc(u.status) << "\","
      << "\"last_seen\":" << u.last_seen
      << "}";
    return o.str();
}

inline std::string whitelist(const UuidWhitelist& w) {
    std::ostringstream o;
    o << "{"
      << "\"uuid\":\""   << jutil::esc(w.uuid) << "\","
      << "\"bearer\":"   << w.bearer           << ","
      << "\"added_at\":" << w.added_at
      << "}";
    return o.str();
}

inline std::string sensor(const Sensor& s) {
    std::ostringstream o;
    o << "{"
      << "\"id\":"          << s.id           << ","
      << "\"node_id\":\""   << jutil::esc(s.node_id) << "\","
      << "\"sensor_id\":"   << s.sensor_id    << ","
      << "\"temperature\":" << s.temperature  << ","
      << "\"humidity\":"    << s.humidity     << ","
      << "\"lux\":"         << s.lux          << ","
      << "\"motion\":"      << s.motion       << ","
      << "\"battery\":"     << s.battery      << ","
      << "\"status\":"      << s.status       << ","
      << "\"timestamp\":"   << s.timestamp
      << "}";
    return o.str();
}

inline std::string actuator(const Actuator& a) {
    std::ostringstream o;
    o << "{"
      << "\"id\":"          << a.id                 << ","
      << "\"node_id\":\""   << jutil::esc(a.node_id)<< "\","
      << "\"actuator_id\":" << a.actuator_id        << ","
      << "\"setpoint\":"    << a.setpoint           << ","
      << "\"onoff\":"       << a.onoff              << ","
      << "\"status\":\""    << jutil::esc(a.status) << "\","
      << "\"issued_at\":"   << a.issued_at          << ","
      << "\"acked_at\":"    << a.acked_at
      << "}";
    return o.str();
}

inline std::string eventlog(const EventLog& e) {
    std::ostringstream o;
    o << "{"
      << "\"id\":"        << e.id                  << ","
      << "\"node_id\":\"" << jutil::esc(e.node_id) << "\","
      << "\"level\":\""   << jutil::esc(e.level)   << "\","
      << "\"event\":\""   << jutil::esc(e.event)   << "\","
      << "\"payload\":"   << (e.payload.empty() ? "{}" : e.payload) << ","
      << "\"timestamp\":" << e.timestamp
      << "}";
    return o.str();
}

inline std::string group(const DeviceGroup& g) {
    std::ostringstream o;
    o << "{"
      << "\"id\":"            << g.id                     << ","
      << "\"node_id\":\""     << jutil::esc(g.node_id)    << "\","
      << "\"element_addr\":"  << g.element_addr           << ","
      << "\"group_addr\":"    << g.group_addr             << ","
      << "\"model_id\":"      << g.model_id               << ","
      << "\"company_id\":"    << g.company_id             << ","
      << "\"sub_or_pub\":\"" << jutil::esc(g.sub_or_pub)  << "\","
      << "\"created_at\":"    << g.created_at
      << "}";
    return o.str();
}

template<typename T>
inline std::string array(const std::vector<T>& vec,
                          std::function<std::string(const T&)> fn) {
    std::ostringstream o;
    o << "[";
    for (size_t i = 0; i < vec.size(); ++i) {
        if (i) o << ",";
        o << fn(vec[i]);
    }
    o << "]";
    return o.str();
}

} // namespace to_json

// ─────────────────────────────────────────────────────────────────────────────
//  DbTranslator
// ─────────────────────────────────────────────────────────────────────────────
class DbTranslator {
public:

    // =========================================================================
    //  A) SIGNAL HANDLERS  (GatewayService → Database)
    //     Nhận IpcMessage có payload = JSON string
    // =========================================================================

    static IpcHandler make_save_device_handler(
        DeviceRepository& repo,
        UnprovisionedDeviceRepository& unprov_repo)
    {
        return [&repo, &unprov_repo](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = DeviceProvisionedDto::from_ipc(msg);
                if (dto.node_id.empty()) return {};

                Device d;
                d.node_id   = dto.node_id;
                d.uuid      = dto.uuid;
                d.net_idx   = dto.net_idx;
                d.elem_num  = dto.elem_num;
                d.unicast   = dto.unicast;
                d.status    = "active";
                d.last_seen = static_cast<int64_t>(time(nullptr));
                if (!dto.node_type.empty()) d.type = dto.node_type;
                repo.upsert(d);

                if (!dto.uuid.empty()) {
                    try { unprov_repo.updateStatus(dto.uuid, "provisioned"); }
                    catch (...) {}
                }
                std::cout << "[DB] AddNewDevice: " << d.node_id
                          << " unicast=0x" << std::hex << d.unicast << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[DB] AddNewDevice error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_save_sensor_handler(SensorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = SensorDto::from_ipc(msg);
                if (dto.node_id.empty()) return {};

                Sensor s;
                s.node_id     = dto.node_id;
                s.sensor_id   = dto.sensor_id;
                s.temperature = dto.temperature;
                s.humidity    = dto.humidity;
                s.lux         = dto.lux;
                s.motion      = dto.motion;
                s.battery     = dto.battery;
                s.status      = dto.status;
                s.timestamp   = static_cast<int64_t>(time(nullptr));
                repo.insert(s);
                std::cout << "[DB] SensorData: " << s.node_id
                          << " T=" << s.temperature << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[DB] SensorData error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_save_actuator_ack_handler(ActuatorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ActuatorAckDto::from_ipc(msg);
                auto pendings = repo.findPending(dto.node_id);
                for (const auto& a : pendings) {
                    if (a.actuator_id == dto.actuator_id) {
                        repo.acknowledge(a.id);
                        std::cout << "[DB] ActuatorAck: id=" << a.id << "\n";
                        return {};
                    }
                }
                if (!pendings.empty()) {
                    repo.acknowledge(pendings.front().id);
                }
            } catch (const std::exception& e) {
                std::cerr << "[DB] ActuatorAck error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_save_device_status_handler(DeviceRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = DeviceStatusDto::from_ipc(msg);
                if (dto.node_id.empty()) return {};
                repo.updateStatus(dto.node_id, dto.is_online ? "active" : "offline");
                if (dto.features) repo.updateFeatures(dto.node_id, dto.features);
            } catch (const std::exception& e) {
                std::cerr << "[DB] DeviceStatus error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_save_unprov_handler(
        UnprovisionedDeviceRepository& repo)
    {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = UnprovAdvDto::from_ipc(msg);
                if (dto.uuid.empty()) return {};
                UnprovisionedDevice u;
                u.uuid      = dto.uuid;
                u.mac       = dto.mac;
                u.rssi      = dto.rssi;
                u.bearer    = dto.bearer;
                u.oob_info  = dto.oob_info;
                u.status    = "seen";
                u.last_seen = static_cast<int64_t>(time(nullptr));
                repo.upsert(u);
            } catch (const std::exception& e) {
                std::cerr << "[DB] UnprovAdv error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_save_mesh_event_handler(
        EventLogRepository&             elog_repo,
        DeviceRepository&               dev_repo,
        GroupRepository&                grp_repo,
        UnprovisionedDeviceRepository& unprov_repo)
    {
        return [&elog_repo, &dev_repo, &grp_repo, &unprov_repo]
               (const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = EventLogDto::from_ipc(msg);

                EventLog e;
                e.node_id   = dto.node_id.empty() ? "gateway" : dto.node_id;
                e.level     = dto.level;
                e.event     = dto.event;
                e.payload   = dto.payload_json;
                e.timestamp = static_cast<int64_t>(time(nullptr));

                // Ensure device exists (avoid FK constraint)
                if (!dev_repo.findById(e.node_id)) {
                    Device d;
                    d.node_id   = e.node_id;
                    d.name      = (e.node_id == "gateway") ? "Gateway" : "AutoNode";
                    d.status    = "active";
                    d.last_seen = e.timestamp;
                    dev_repo.upsert(d);
                }
                elog_repo.insert(e);

                _handle_side_effects(e, dev_repo, grp_repo, unprov_repo);
            } catch (const std::exception& ex) {
                std::cerr << "[DB] MeshEvent error: " << ex.what() << "\n";
            }
            return {};
        };
    }

    // =========================================================================
    //  B) METHOD HANDLERS  (D-Bus query → JSON response)
    //     Trả về IpcMessage với payload = JSON string
    //     Format: { "data": [...] }  hoặc  { "error": "..." }
    // =========================================================================

    // ── Device ────────────────────────────────────────────────────────────────

    static IpcHandler make_device_findall_handler(DeviceRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string gw   = jutil::get_str(json, "gateway_id");
                auto devs = repo.findAll(gw);
                return dbt_resp::ok(to_json::array<Device>(devs,
                    [](const Device& d){ return to_json::device(d); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_device_findbyid_handler(DeviceRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string nid  = jutil::get_str(json, "node_id");
                auto opt = repo.findById(nid);
                if (!opt) return dbt_resp::err("not_found");
                return dbt_resp::ok(to_json::device(*opt));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    // ── Unprovisioned ─────────────────────────────────────────────────────────

    static IpcHandler make_unprov_findall_handler(
        UnprovisionedDeviceRepository& repo)
    {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json   = jutil::from_gvariant(msg.payload);
                std::string status = jutil::get_str(json, "status");
                auto rows = repo.findAll(status);
                return dbt_resp::ok(to_json::array<UnprovisionedDevice>(rows,
                    [](const UnprovisionedDevice& u){ return to_json::unprov(u); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    // ── UUID Whitelist ─────────────────────────────────────────────────────────

    static IpcHandler make_whitelist_findall_handler(UuidWhitelistRepository& repo) {
        return [&repo](const IpcMessage& /*msg*/) -> IpcMessage {
            try {
                auto rows = repo.findAll();
                return dbt_resp::ok(to_json::array<UuidWhitelist>(rows,
                    [](const UuidWhitelist& w){ return to_json::whitelist(w); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_whitelist_add_handler(UuidWhitelistRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = UuidWhitelistDto::from_ipc(msg);
                UuidWhitelist w;
                w.uuid     = dto.uuid;
                w.bearer   = dto.bearer;
                w.added_at = static_cast<int64_t>(time(nullptr));
                repo.add(w);
                return dbt_resp::ok_flag();
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_whitelist_remove_handler(UuidWhitelistRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string uuid = jutil::get_str(json, "uuid");
                if (uuid.empty()) return dbt_resp::err("missing uuid");
                repo.remove(uuid);
                return dbt_resp::ok_flag();
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    // ── Sensor ─────────────────────────────────────────────────────────────────

    static IpcHandler make_sensor_findlatest_handler(SensorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string nid  = jutil::get_str(json, "node_id");
                auto opt = repo.findLatest(nid);
                if (!opt) return dbt_resp::err("not_found");
                return dbt_resp::ok(to_json::sensor(*opt));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_sensor_findhistory_handler(SensorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string nid  = jutil::get_str(json, "node_id");
                int64_t from     = jutil::get_int64(json, "from");
                int64_t to       = jutil::get_int64(json, "to",
                    static_cast<int64_t>(time(nullptr)));
                int limit        = jutil::get_int(json, "limit", 100);
                auto rows = repo.findHistory(nid, from, to, limit);
                return dbt_resp::ok(to_json::array<Sensor>(rows,
                    [](const Sensor& s){ return to_json::sensor(s); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_sensor_findall_handler(SensorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                int limit        = jutil::get_int(json, "limit", 100);
                auto rows = repo.findAll(limit);
                return dbt_resp::ok(to_json::array<Sensor>(rows,
                    [](const Sensor& s){ return to_json::sensor(s); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    // ── Actuator ───────────────────────────────────────────────────────────────

    static IpcHandler make_actuator_findall_handler(ActuatorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string nid  = jutil::get_str(json, "node_id");
                auto rows = repo.findAll(nid);
                return dbt_resp::ok(to_json::array<Actuator>(rows,
                    [](const Actuator& a){ return to_json::actuator(a); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_actuator_findbyid_handler(ActuatorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                int64_t id       = jutil::get_int64(json, "id");
                auto opt = repo.findById(id);
                if (!opt) return dbt_resp::err("not_found");
                return dbt_resp::ok(to_json::actuator(*opt));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_actuator_findpending_handler(ActuatorRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                std::string nid  = jutil::get_str(json, "node_id");
                auto rows = repo.findPending(nid);
                return dbt_resp::ok(to_json::array<Actuator>(rows,
                    [](const Actuator& a){ return to_json::actuator(a); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    // ── EventLog ───────────────────────────────────────────────────────────────

    static IpcHandler make_eventlog_findall_handler(EventLogRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json  = jutil::from_gvariant(msg.payload);
                std::string nid   = jutil::get_str(json, "node_id");
                std::string level = jutil::get_str(json, "level");
                int limit         = jutil::get_int(json, "limit", 200);
                auto rows = repo.findAll(nid, level, limit);
                return dbt_resp::ok(to_json::array<EventLog>(rows,
                    [](const EventLog& e){ return to_json::eventlog(e); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_eventlog_findbyid_handler(EventLogRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                int64_t id       = jutil::get_int64(json, "id");
                auto opt = repo.findById(id);
                if (!opt) return dbt_resp::err("not_found");
                return dbt_resp::ok(to_json::eventlog(*opt));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    // ── Group ──────────────────────────────────────────────────────────────────

    static IpcHandler make_group_findall_handler(GroupRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json       = jutil::from_gvariant(msg.payload);
                std::string nid        = jutil::get_str(json, "node_id");
                std::string sub_or_pub = jutil::get_str(json, "sub_or_pub");
                auto rows = repo.findAll(nid, sub_or_pub);
                return dbt_resp::ok(to_json::array<DeviceGroup>(rows,
                    [](const DeviceGroup& g){ return to_json::group(g); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_group_findbyid_handler(GroupRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json = jutil::from_gvariant(msg.payload);
                int64_t id       = jutil::get_int64(json, "id");
                auto opt = repo.findById(id);
                if (!opt) return dbt_resp::err("not_found");
                return dbt_resp::ok(to_json::group(*opt));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

    static IpcHandler make_group_findbyaddr_handler(GroupRepository& repo) {
        return [&repo](const IpcMessage& msg) -> IpcMessage {
            try {
                std::string json       = jutil::from_gvariant(msg.payload);
                int group_addr         = jutil::get_int(json, "group_addr");
                std::string sub_or_pub = jutil::get_str(json, "sub_or_pub");
                auto rows = repo.findByGroupAddr(group_addr, sub_or_pub);
                return dbt_resp::ok(to_json::array<DeviceGroup>(rows,
                    [](const DeviceGroup& g){ return to_json::group(g); }));
            } catch (const std::exception& e) { return dbt_resp::err(e.what()); }
        };
    }

private:
    static void _handle_side_effects(
        const EventLog&                e,
        DeviceRepository&              dev_repo,
        GroupRepository&               grp_repo,
        UnprovisionedDeviceRepository& unprov_repo)
    {
        const std::string& ev = e.event;
        const std::string& pl = e.payload;

        if ((ev == "ProvComplete" || ev == "RprProvComplete")
            && !e.node_id.empty() && e.node_id != "gateway") {
            dev_repo.updateStatus(e.node_id, "active");
        }
        else if (ev == "BindAppKeyComp"
                 && pl.find("\"node_ready\":true") != std::string::npos) {
            dev_repo.updateStatus(e.node_id, "active");
        }
        else if (ev == "NodeReset" && !e.node_id.empty()) {
            dev_repo.deprovision(e.node_id);
        }
        else if (ev == "RelayStatus") {
            int rs = jutil::get_int(pl, "relay_state");
            try { dev_repo.updateRelay(e.node_id, rs); } catch (...) {}
        }
        else if (ev == "ProxyStatus") {
            try { dev_repo.updateProxy(e.node_id, jutil::get_int(pl, "state")); }
            catch (...) {}
        }
        else if (ev == "FriendStatus") {
            try { dev_repo.updateFriend(e.node_id, jutil::get_int(pl, "state")); }
            catch (...) {}
        }
        else if ((ev == "ModelSubscribeStatus" || ev == "GroupAddComp")
                 && jutil::get_int(pl, "status") == 0) {
            DeviceGroup g;
            g.node_id    = e.node_id;
            g.group_addr = jutil::get_int(pl, "group_addr");
            g.model_id   = jutil::get_int(pl, "model_id");
            g.company_id = jutil::get_int(pl, "company_id", 0xFFFF);
            g.sub_or_pub = "sub";
            try { grp_repo.insert(g); } catch (...) {}
        }
        else if ((ev == "ModelUnsubscribeStatus" || ev == "GroupRemoveComp")
                 && jutil::get_int(pl, "status") == 0) {
            try {
                grp_repo.removeByKey(e.node_id, 0,
                    jutil::get_int(pl, "group_addr"),
                    jutil::get_int(pl, "model_id"),
                    jutil::get_int(pl, "company_id", 0xFFFF), "sub");
            } catch (...) {}
        }
        else if (ev == "ModelPublishStatus"
                 && jutil::get_int(pl, "status") == 0) {
            DeviceGroup g;
            g.node_id    = e.node_id;
            g.group_addr = jutil::get_int(pl, "pub_addr");
            g.model_id   = jutil::get_int(pl, "model_id");
            g.company_id = jutil::get_int(pl, "company_id", 0xFFFF);
            g.sub_or_pub = "pub";
            try { grp_repo.insert(g); } catch (...) {}
        }
        else if (ev == "SetNodeNameComp") {
            std::string name = jutil::get_str(pl, "name");
            if (!name.empty()) {
                try { dev_repo.updateName(e.node_id, name); } catch (...) {}
            }
        }
    }
};

#endif // DB_TRANSLATOR_H

#ifndef GATEWAY_TRANSLATOR_H
#define GATEWAY_TRANSLATOR_H

/**
 * @file gateway_translator.h
 *
 * Tách biệt hoàn toàn logic parse IpcMessage -> DTO -> MeshCommandSender.
 * Giúp GatewayService gọn gàng và dễ test hơn.
 */

#include "ipc_message.h"
#include "ipc_dto.h"                  
#include "mesh_command_sender.hpp"        
#include "uart_msgs_def.h"                
#include <functional>
#include <mutex>
#include <unordered_set>
#include <iostream>
#include <sstream>
#include <iomanip>
#include <string>

// ── Helper nội bộ ────────────────────────────────────────────────────────
namespace {
    inline bool hex_to_bytes(const std::string& hex, uint8_t* out, size_t len) {
        if (hex.size() < len * 2) return false;
        for (size_t i = 0; i < len; ++i)
            out[i] = static_cast<uint8_t>(std::stoul(hex.substr(i*2, 2), nullptr, 16));
        return true;
    }

    inline uint16_t unicast_from_node_id(const std::string& node_id) {
        if (node_id.rfind("node_", 0) == 0 && node_id.size() > 5) {
            try {
                return static_cast<uint16_t>(std::stoul(node_id.substr(5), nullptr, 16));
            } catch (...) {}
        }
        return 0;
    }
} // namespace

class GatewayTranslator {
public:
    using IpcHandler = std::function<IpcMessage(const IpcMessage&)>;

    // ── PROVISIONING ─────────────────────────────────────────────────────

    static IpcHandler make_provision_start_handler(
        MeshCommandSender& sender,
        std::mutex& known_uuids_mutex,
        std::unordered_set<std::string>& known_uuids)
    {
        return [&sender, &known_uuids_mutex, &known_uuids](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<ProvisionStartDto>(msg);
                sender.prov_enable();

                if (dto.uuid_hex.empty()) {
                    std::cout << "[GW] IPC ProvisionStart → enable scan (all)\n";
                } else {
                    std::lock_guard<std::mutex> lk(known_uuids_mutex);
                    if (known_uuids.count(dto.uuid_hex) == 0) {
                        known_uuids.insert(dto.uuid_hex);
                        uint8_t uuid[16] = {0};
                        if (hex_to_bytes(dto.uuid_hex, uuid, 16)) {
                            sender.add_unprov_dev(uuid, static_cast<uint8_t>(dto.bearer));
                            std::cout << "[GW] IPC ProvisionStart → uuid=" << dto.uuid_hex << "\n";
                        }
                    } else {
                        std::cout << "[GW] IPC ProvisionStart → uuid=" << dto.uuid_hex << " already known\n";
                    }
                }
            } catch (const std::exception& e) {
                std::cerr << "[GW] SendProvisionStart error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_provision_stop_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage&) -> IpcMessage {
            try {
                sender.prov_disable();
                std::cout << "[GW] IPC ProvisionStop\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] SendProvisionStop error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_set_uuid_match_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<UuidWhitelistDto>(msg);
                uint8_t match[8] = {0};
                hex_to_bytes(dto.uuid, match, 8);
                sender.set_dev_uuid_match(match);
                std::cout << "[GW] IPC SetUuidMatch → " << dto.uuid << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] SetUuidMatch error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_delete_node_handler(
        MeshCommandSender& sender,
        std::mutex& known_uuids_mutex,
        std::unordered_set<std::string>& known_uuids)
    {
        return [&, &sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<DeleteNodeDto>(msg);
                uint16_t uni = unicast_from_node_id(dto.node_id);
                if (uni == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                sender.delete_node(uni);
                {
                    std::lock_guard<std::mutex> lk(known_uuids_mutex);
                    known_uuids.erase(dto.uuid_hex);
                }
                std::cout << "[GW] IPC DeleteNode → 0x" << std::hex << uni << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] DeleteNode error: " << e.what() << "\n";
            }
            return {};
        };
    }

    // ── ACTUATOR COMMANDS ────────────────────────────────────────────────

    static IpcHandler make_actuator_cmd_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<ActuatorDto>(msg);
                uint16_t uni = unicast_from_node_id(dto.node_id);
                if (uni == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);

                uint16_t sp  = static_cast<uint16_t>(dto.setpoint);
                uint8_t  st  = dto.onoff ? 1 : 0;
                sender.actuator_set(uni, static_cast<uint16_t>(dto.actuator_id), sp, st);
                std::cout << "[GW] IPC SendActuatorCmd → node=" << dto.node_id << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] SendActuatorCmd error: " << e.what() << "\n";
            }
            return {};
        };
    }

    // ── GROUP OPERATIONS ─────────────────────────────────────────────────

    static IpcHandler make_subscribe_group_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<SubscribeGroupDto>(msg);
                uint16_t uni = unicast_from_node_id(dto.node_id);
                if (uni == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                sender.group_add(uni, dto.element_addr, dto.group_addr, dto.model_id, dto.company_id);
                std::cout << "[GW] IPC SubscribeToGroup → node=0x" << std::hex << uni << " group=0x" << dto.group_addr << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] SubscribeToGroup error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_publish_group_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<PublishGroupDto>(msg);
                uint16_t uni = unicast_from_node_id(dto.node_id);
                if (uni == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                sender.model_pub_set(uni, dto.element_addr, dto.pub_addr, dto.model_id, dto.company_id, dto.ttl, dto.period);
                std::cout << "[GW] IPC PublishToGroup → node=0x" << std::hex << uni << " pub=0x" << dto.pub_addr << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] PublishToGroup error: " << e.what() << "\n";
            }
            return {};
        };
    }

    // ── RPR (Remote Provisioning) ────────────────────────────────────────

    static IpcHandler make_rpr_scan_start_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<RprScanStartDto>(msg);
                uint16_t srv_addr = unicast_from_node_id(dto.node_id);
                if (srv_addr == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                uint8_t uuid[16] = {0};
                if (dto.uuid_filter_en && !dto.uuid.empty()) {
                    hex_to_bytes(dto.uuid, uuid, 16);
                }
                sender.rpr_scan_start(srv_addr, dto.limit, dto.timeout, dto.uuid_filter_en, uuid);
                std::cout << "[GW] IPC RprScanStart → srv=0x" << std::hex << srv_addr << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] RprScanStart error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_rpr_link_open_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<RprLinkOpenDto>(msg);
                uint16_t srv_addr = unicast_from_node_id(dto.node_id);
                if (srv_addr == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                uint8_t uuid[16] = {0};
                if (!dto.uuid.empty()) hex_to_bytes(dto.uuid, uuid, 16);
                sender.rpr_link_open(srv_addr, uuid, dto.timeout_en, dto.timeout);
                std::cout << "[GW] IPC RprLinkOpen → srv=0x" << std::hex << srv_addr << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] RprLinkOpen error: " << e.what() << "\n";
            }
            return {};
        };
    }

    // ── HEARTBEAT ────────────────────────────────────────────────────────

    static IpcHandler make_heartbeat_pub_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<HeartbeatPubDto>(msg);
                uint16_t uni = unicast_from_node_id(dto.node_id);
                if (uni == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                sender.heartbeat_pub(uni, dto.dst, dto.period, dto.ttl);
                std::cout << "[GW] IPC HeartbeatPub → node=0x" << std::hex << uni << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] HeartbeatPub error: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_heartbeat_sub_handler(MeshCommandSender& sender) {
        return [&sender](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = ipc_to_dto<HeartbeatSubDto>(msg);
                uint16_t uni = unicast_from_node_id(dto.node_id);
                if (uni == 0) throw std::invalid_argument("Invalid node_id: " + dto.node_id);
                
                sender.heartbeat_sub(uni, dto.group_addr);
                std::cout << "[GW] IPC HeartbeatSub → node=0x" << std::hex << uni << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[GW] HeartbeatSub error: " << e.what() << "\n";
            }
            return {};
        };
    }
};

#endif // GATEWAY_TRANSLATOR_H

#ifndef _IPC_MESSAGE_H
#define _IPC_MESSAGE_H
#include <string>
#include <stdint.h>
#include <map>
#include "gio/gio.h"
struct IpcMessage {
    GVariant* payload = nullptr; 
};

#endif //_IPC_MESSAGE_H

#ifndef IPC_H
#define IPC_H

#include <string>
#include <functional>
#include <map>
#include <vector>
#include "../ipc_translator/ipc_message.h"

enum class DataStatus {
    OK = 0,
    ERROR,
    NOT_READY
};

using IpcHandler = std::function<IpcMessage(const IpcMessage& payload)>;

struct DBusConfig {
    std::string serviceName;
    std::string objectPath;
    std::string interfaceName;

    static DBusConfig forMesh() { return {"com.gateway.mesh", "/com/gateway/mesh", "com.gateway.mesh.interface"}; }
    static DBusConfig forDatabase() { return {"com.gateway.database", "/com/gateway/database", "com.gateway.database.interface"}; }
    static DBusConfig forMqtt() { return {"com.gateway.mqtt", "/com/gateway/mqtt", "com.gateway.mqtt.interface"}; }
    static DBusConfig forUI() { return {"com.gateway.ui", "/com/gateway/ui", "com.gateway.ui.interface"}; }
};

class IIpc {
public:
    virtual ~IIpc() = default;

    virtual DataStatus init()    = 0;
    virtual DataStatus de_init() = 0;
    virtual DataStatus send(const IpcMessage& data, std::string signalName) = 0;
    virtual DataStatus process() = 0;

    virtual DataStatus call_method(
        const std::string& dest_svc, const std::string& dest_path,
        const std::string& dest_iface, const std::string& method,
        const IpcMessage&  payload, std::vector<IpcMessage>& results) = 0;

    virtual void subscribe(
        const std::string& svc, const std::string& path,
        const std::string& iface, const std::string& signal,
        IpcHandler         handler) = 0;

    virtual void expose_method(const std::string& method, IpcHandler handler) = 0;
};

#endif

#ifndef MQTT_TRANSLATOR_H
#define MQTT_TRANSLATOR_H

/**
 * @file mqtt_translator.h  (MqttApp)
 *
 * Dịch giữa D-Bus JSON IPC ↔ MQTT topic/payload JSON.
 * Không còn dùng GVariant typed tuple.
 *
 * Topic schema:
 *   gateway/nodes/<node_id>/status        ← AddNewDevice, DeviceStatus
 *   gateway/nodes/<node_id>/sensors       ← SensorData
 *   gateway/nodes/<node_id>/actuator/ack  ← ActuatorAck
 *   gateway/nodes/<node_id>/events        ← MeshEvent
 *   gateway/scan/unprov_adv               ← UnprovAdv
 *   gateway/nodes/+/command               ← (subscribe) cloud commands
 */

#include "ipc_dto.h"
#include "ipc_message.h"
#include "ipc_json_utils.h"
#include "mqtt_message.h"
#include "mqtt_client.h"
#include "gdbus.h"

#include <string>
#include <iostream>

// ─────────────────────────────────────────────────────────────────────────────
//  Topic helpers
// ─────────────────────────────────────────────────────────────────────────────
namespace MqttTopic {
    inline std::string node_status  (const std::string& id) { return "gateway/nodes/" + id + "/status";       }
    inline std::string node_sensors (const std::string& id) { return "gateway/nodes/" + id + "/sensors";      }
    inline std::string node_actuator(const std::string& id) { return "gateway/nodes/" + id + "/actuator/ack"; }
    inline std::string node_events  (const std::string& id) { return "gateway/nodes/" + id + "/events";       }
    inline std::string unprov_adv   ()                      { return "gateway/scan/unprov_adv";               }
    inline std::string cmd_filter   ()                      { return "gateway/nodes/+/command";               }
}

// ─────────────────────────────────────────────────────────────────────────────
//  MqttTranslator
// ─────────────────────────────────────────────────────────────────────────────
class MqttTranslator {
public:
    static MQTTMessage provisioned_to_mqtt(const DeviceProvisionedDto& dto) {
        std::string base = dto.to_json();
        std::string payload = "{\"event\":\"provisioned\"," + base.substr(1);

        MQTTMessage msg;
        msg.topic   = MqttTopic::node_status(dto.node_id);
        msg.payload = payload;
        msg.qos     = QOS_1_AT_LEAST_ONCE;
        msg.retain  = true;
        return msg;
    }

    /**
     * DeviceStatus → gateway/nodes/<node_id>/status
     */
    static MQTTMessage device_status_to_mqtt(const DeviceStatusDto& dto) {
        std::string ev = dto.is_online ? "heartbeat" : "offline";
        std::string base = dto.to_json();
        std::string payload = "{\"event\":\"" + ev + "\"," + base.substr(1);

        MQTTMessage msg;
        msg.topic   = MqttTopic::node_status(dto.node_id);
        msg.payload = payload;
        msg.qos     = QOS_1_AT_LEAST_ONCE;
        msg.retain  = true;
        return msg;
    }

    /**
     * SensorData → gateway/nodes/<node_id>/sensors
     * QoS 0 — continuous data
     */
    static MQTTMessage sensor_to_mqtt(const SensorDto& dto) {
        MQTTMessage msg;
        msg.topic   = MqttTopic::node_sensors(dto.node_id);
        msg.payload = dto.to_json();
        msg.qos     = QOS_0_AT_MOST_ONCE;
        msg.retain  = false;
        return msg;
    }

    /**
     * ActuatorAck → gateway/nodes/<node_id>/actuator/ack
     */
    static MQTTMessage actuator_ack_to_mqtt(const ActuatorAckDto& dto) {
        MQTTMessage msg;
        msg.topic   = MqttTopic::node_actuator(dto.node_id);
        msg.payload = dto.to_json();
        msg.qos     = QOS_1_AT_LEAST_ONCE;
        msg.retain  = false;
        return msg;
    }

    /**
     * MeshEvent → gateway/nodes/<node_id>/events
     */
    static MQTTMessage mesh_event_to_mqtt(const EventLogDto& dto) {
        const std::string& nid = dto.node_id.empty() ? "gateway" : dto.node_id;
        MQTTMessage msg;
        msg.topic   = MqttTopic::node_events(nid);
        msg.payload = dto.to_json();
        msg.qos     = QOS_0_AT_MOST_ONCE;
        msg.retain  = false;
        return msg;
    }

    /**
     * UnprovAdv → gateway/scan/unprov_adv
     */
    static MQTTMessage unprov_adv_to_mqtt(const UnprovAdvDto& dto) {
        MQTTMessage msg;
        msg.topic   = MqttTopic::unprov_adv();
        msg.payload = dto.to_json();
        msg.qos     = QOS_0_AT_MOST_ONCE;
        msg.retain  = false;
        return msg;
    }

    // ── IpcHandler factories ──────────────────────────────────────────────────

    static IpcHandler make_provisioned_handler(MQTTClient& client) {
        return [&client](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = DeviceProvisionedDto::from_ipc(msg);
                auto mqtt = provisioned_to_mqtt(dto);
                client.publish(mqtt);
                std::cout << "[MQTT] AddNewDevice → " << mqtt.topic << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] provisioned_handler: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_device_status_handler(MQTTClient& client) {
        return [&client](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto = DeviceStatusDto::from_ipc(msg);
                auto mqtt = device_status_to_mqtt(dto);
                client.publish(mqtt);
                std::cout << "[MQTT] DeviceStatus → " << mqtt.topic
                          << " (" << (dto.is_online ? "online" : "offline") << ")\n";
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] device_status_handler: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_sensor_handler(MQTTClient& client) {
        return [&client](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto  = SensorDto::from_ipc(msg);
                auto mqtt = sensor_to_mqtt(dto);
                client.publish(mqtt);
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] sensor_handler: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_actuator_ack_handler(MQTTClient& client) {
        return [&client](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto  = ActuatorAckDto::from_ipc(msg);
                auto mqtt = actuator_ack_to_mqtt(dto);
                client.publish(mqtt);
                std::cout << "[MQTT] ActuatorAck → " << mqtt.topic << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] actuator_ack_handler: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_mesh_event_handler(MQTTClient& client) {
        return [&client](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto  = EventLogDto::from_ipc(msg);
                auto mqtt = mesh_event_to_mqtt(dto);
                client.publish(mqtt);
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] mesh_event_handler: " << e.what() << "\n";
            }
            return {};
        };
    }

    static IpcHandler make_unprov_adv_handler(MQTTClient& client) {
        return [&client](const IpcMessage& msg) -> IpcMessage {
            try {
                auto dto  = UnprovAdvDto::from_ipc(msg);
                auto mqtt = unprov_adv_to_mqtt(dto);
                client.publish(mqtt);
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] unprov_adv_handler: " << e.what() << "\n";
            }
            return {};
        };
    }

    // ── Cloud command forwarder: MQTT → D-Bus ─────────────────────────────────
    /**
     * Topic: gateway/nodes/<node_id>/command
     * Payload JSON: { "actuator_id": 1, "onoff": true, "setpoint": 75.0 }
     * → Forward as ActuatorDto to GatewayService "SendActuatorCmd"
     */
    static std::function<void(const MQTTMessage&)>
    make_command_forwarder(DBusGDBus& dbus) {
        return [&dbus](const MQTTMessage& mqtt_msg) {
            if (mqtt_msg.topic.find("/command") == std::string::npos) return;

            std::string node_id = _extract_node_id(mqtt_msg.topic);
            if (node_id.empty()) {
                std::cerr << "[MQTT] command: cannot parse node_id from "
                          << mqtt_msg.topic << "\n";
                return;
            }

            try {
                // Build ActuatorDto from MQTT payload JSON
                ActuatorDto dto;
                dto.node_id     = node_id;
                dto.actuator_id = jutil::get_int   (mqtt_msg.payload, "actuator_id");
                dto.onoff       = jutil::get_bool  (mqtt_msg.payload, "onoff");
                dto.setpoint    = jutil::get_double(mqtt_msg.payload, "setpoint");
                dto.status      = "pending";

                std::vector<IpcMessage> results;
                dbus.call_method(
                    "com.gateway.mesh",
                    "/com/gateway/mesh",
                    "com.gateway.mesh.interface",
                    "SendActuatorCmd",
                    dto.to_ipc(), results);

                std::cout << "[MQTT] Command forwarded → node=" << node_id
                          << " act=" << dto.actuator_id
                          << " setpoint=" << dto.setpoint
                          << " onoff=" << dto.onoff << "\n";
            } catch (const std::exception& e) {
                std::cerr << "[MQTT] command_forwarder: " << e.what() << "\n";
            }
        };
    }

private:
    // "gateway/nodes/node_0001/command" → "node_0001"
    static std::string _extract_node_id(const std::string& topic) {
        const std::string prefix = "nodes/";
        auto start = topic.find(prefix);
        if (start == std::string::npos) return "";
        start += prefix.size();
        auto end = topic.find('/', start);
        return topic.substr(start,
            end == std::string::npos ? std::string::npos : end - start);
    }
};

#endif // MQTT_TRANSLATOR_H
