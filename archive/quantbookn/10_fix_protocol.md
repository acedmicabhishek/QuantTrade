# FIX Protocol: Session and Application Layer

FIX (Financial Information eXchange) is the universal protocol for electronic order entry across brokers, prime brokers, and buy-side firms. While slower than binary protocols (OUCH/SBE), FIX is ubiquitous — every execution desk uses it. A QD must understand both the protocol mechanics and its production implementation.

---

## 1. FIX Message Structure

Every FIX message is a sequence of tag=value pairs separated by SOH (byte `0x01`, displayed as `|`).

```
8=FIX.4.2|9=112|35=D|49=SENDER_COMP|56=TARGET_COMP|34=1001|52=20241105-14:23:01.123|
11=ORD-001|21=1|55=AAPL|54=1|38=100|40=2|44=182.50|59=0|10=134|
```

### 1.1 Standard Header Fields

| Tag | Field Name | Description |
|-----|------------|-------------|
| 8   | BeginString | FIX version: `FIX.4.2`, `FIX.4.4`, `FIXT.1.1` |
| 9   | BodyLength | Byte count from tag 35 to tag 10 (exclusive) |
| 35  | MsgType | Message type: `D`=NewOrderSingle, `8`=ExecutionReport |
| 49  | SenderCompID | Sender identifier |
| 56  | TargetCompID | Receiver identifier |
| 34  | MsgSeqNum | Monotonic sequence number per session |
| 52  | SendingTime | UTC timestamp: `YYYYMMDD-HH:MM:SS.sss` |

### 1.2 Standard Trailer

| Tag | Field Name | Description |
|-----|------------|-------------|
| 10  | Checksum | Sum of all bytes mod 256, formatted as 3-digit string |

Checksum computation:
```cpp
std::string compute_checksum(const std::string& msg) {
    uint32_t sum = 0;
    for (unsigned char c : msg) sum += c;
    char buf[4];
    snprintf(buf, sizeof(buf), "%03d", sum % 256);
    return buf;
}
```

---

## 2. Session Layer

FIX sessions manage TCP connections, sequence numbers, and message recovery. A session must be maintained continuously — sequence gaps trigger retransmission.

### 2.1 Session State Machine

```
DISCONNECTED
     │ connect()
     ▼
CONNECTED
     │ send Logon (35=A)
     ▼
LOGON_SENT
     │ receive Logon from counterparty
     ▼
ACTIVE  ◄─────────────────────────────┐
     │                                │
     │ receive HeartBeat (35=0)        │ send HeartBeat (35=0)
     │ receive TestRequest (35=1)      │ every HeartBtInt seconds
     │ receive ResendRequest (35=2)    │
     │ receive Reject (35=3)           │
     │                                │
     │ send Logout (35=5) or TCP drop  │
     ▼                                │
LOGOUT_SENT ──────────────────────────┘
     │ receive Logout
     ▼
DISCONNECTED
```

### 2.2 Logon Message

```
8=FIX.4.2|9=68|35=A|49=CLIENT|56=BROKER|34=1|52=20241105-14:23:00.000|
108=30|98=0|10=087|
```

Key fields:
- `35=A`: MsgType = Logon
- `108=30`: HeartBtInt = 30 seconds (both sides agree)
- `98=0`: EncryptMethod = None (FIX encryption is rarely used; TLS at TCP layer)
- `34=1`: First message in session, seq num always starts at 1

### 2.3 HeartBeat and Test Request

```python
class FIXSessionManager:
    def __init__(self, heartbt_int: int = 30):
        self.heartbt_int     = heartbt_int
        self.last_send_time  = 0.0
        self.last_recv_time  = 0.0
        self.seq_num_out     = 1
        self.seq_num_in      = 1

    def on_tick(self, now: float, sender) -> list[str]:
        msgs = []

        # Send heartbeat if idle
        if now - self.last_send_time > self.heartbt_int:
            msgs.append(self._build_heartbeat())

        # Send TestRequest if counterparty silent
        if now - self.last_recv_time > self.heartbt_int * 1.5:
            msgs.append(self._build_test_request())

        # Disconnect if no response to TestRequest after another interval
        if now - self.last_recv_time > self.heartbt_int * 2.5:
            raise FIXSessionTimeout("Counterparty unresponsive")

        return msgs

    def on_message_received(self, msg: dict, now: float) -> list[str]:
        self.last_recv_time = now
        seq = int(msg['34'])
        responses = []

        if seq > self.seq_num_in:
            # Gap detected — request retransmission
            responses.append(self._build_resend_request(self.seq_num_in, seq - 1))
        elif seq < self.seq_num_in:
            # Duplicate — discard
            return []
        else:
            self.seq_num_in += 1

        if msg['35'] == '1':   # TestRequest
            responses.append(self._build_heartbeat(msg.get('112', '')))

        return responses
```

### 2.4 Gap Fill and ResendRequest

When a gap is detected (`seq_num > expected`), send:
```
35=2|7=101|16=110|    (ResendRequest: MsgSeqNum 101 through 110)
```

Counterparty replays messages. **Gap-fill** messages (`35=4`, SequenceReset) are sent for admin messages (heartbeats, logons) that don't need replay.

---

## 3. Application Layer: Key Message Types

### 3.1 New Order Single (35=D)

```
8=FIX.4.2|9=148|35=D|49=BUYSIDE|56=BROKER|34=456|52=20241105-14:30:01.234|
11=ORD-20241105-001|    ← ClOrdID: client order ID (must be unique per session)
21=1|                   ← HandlInst: 1=Automated
55=AAPL|                ← Symbol
54=1|                   ← Side: 1=Buy, 2=Sell, 5=Sell Short
38=1000|                ← OrderQty
40=2|                   ← OrdType: 1=Market, 2=Limit, 3=Stop
44=182.50|              ← Price (required for Limit)
59=0|                   ← TimeInForce: 0=Day, 1=GTC, 3=IOC, 4=FOK
10=221|
```

### 3.2 Execution Report (35=8)

The broker/exchange sends back an ExecutionReport for every state change:

```
35=8|                   ← MsgType: ExecutionReport
11=ORD-20241105-001|    ← ClOrdID: echoed back
37=EXCH-98765|          ← OrderID: exchange-assigned
17=FILL-001|            ← ExecID: unique per execution
150=2|                  ← ExecType: 0=New, 1=PartialFill, 2=Fill, 4=Cancelled, 8=Rejected
39=2|                   ← OrdStatus: 0=New, 1=PartiallyFilled, 2=Filled, 4=Cancelled
55=AAPL|
54=1|
38=1000|                ← OrderQty
32=500|                 ← LastQty: shares filled in this execution
31=182.48|              ← LastPx: fill price
14=500|                 ← CumQty: total filled so far
151=500|                ← LeavesQty: remaining open qty
```

```python
def parse_exec_report(msg: dict) -> dict:
    exec_type = msg['150']
    return {
        'cl_ord_id':    msg['11'],
        'order_id':     msg.get('37'),
        'exec_type':    exec_type,
        'status': {
            '0': 'NEW',
            '1': 'PARTIAL_FILL',
            '2': 'FILL',
            '4': 'CANCELLED',
            '8': 'REJECTED',
        }.get(exec_type, 'UNKNOWN'),
        'last_qty':     int(msg.get('32', 0)),
        'last_px':      float(msg.get('31', 0)),
        'cum_qty':      int(msg['14']),
        'leaves_qty':   int(msg['151']),
        'reject_reason': msg.get('58', ''),
    }
```

### 3.3 Order Cancel Request (35=F)

```
35=F|
11=CANCEL-001|          ← New ClOrdID for the cancel
41=ORD-20241105-001|    ← OrigClOrdID: ID of order to cancel
55=AAPL|
54=1|
38=1000|
10=...|
```

### 3.4 Order Cancel/Replace Request (35=G)

Atomically cancel and replace (modify) an open order:

```
35=G|
11=MODIFY-001|          ← New ClOrdID
41=ORD-20241105-001|    ← OrigClOrdID
55=AAPL|
54=1|
38=800|                 ← New quantity
40=2|
44=183.00|              ← New price
```

---

## 4. Production FIX Implementation

### 4.1 QuickFIX/N Library

QuickFIX/N (C++) is the standard open-source FIX engine. Configure via `fix.cfg`:

```ini
[DEFAULT]
ConnectionType=initiator
HeartBtInt=30
ReconnectInterval=10
FileStorePath=./store
FileLogPath=./log

[SESSION]
BeginString=FIX.4.2
SenderCompID=BUYSIDE_01
TargetCompID=BROKER_GW
SocketConnectHost=fix.broker.com
SocketConnectPort=9876
StartTime=07:00:00
EndTime=22:00:00
```

Strategy callback interface:

```cpp
#include "quickfix/Application.h"
#include "quickfix/fix42/NewOrderSingle.h"
#include "quickfix/fix42/ExecutionReport.h"

class MyFIXApp : public FIX::Application {
public:
    void fromApp(const FIX::Message& msg, const FIX::SessionID& sid) override {
        crack(msg, sid);  // dispatches to typed handlers
    }

    void onMessage(const FIX::FIX42::ExecutionReport& report,
                   const FIX::SessionID&) override {
        FIX::ExecType execType;
        FIX::CumQty cumQty;
        FIX::LastPx lastPx;
        report.get(execType);
        report.get(cumQty);
        report.get(lastPx);
        // Update internal order state
    }

    void send_order(const std::string& symbol, char side,
                    int qty, double price) {
        FIX42::NewOrderSingle order;
        order.set(FIX::ClOrdID("ORD-" + std::to_string(++order_id_)));
        order.set(FIX::Symbol(symbol));
        order.set(FIX::Side(side));
        order.set(FIX::OrderQty(qty));
        order.set(FIX::OrdType(FIX::OrdType_LIMIT));
        order.set(FIX::Price(price));
        order.set(FIX::TimeInForce(FIX::TimeInForce_DAY));
        order.set(FIX::HandlInst('1'));
        order.set(FIX::TransactTime());
        FIX::Session::sendToTarget(order, session_id_);
    }

private:
    FIX::SessionID session_id_;
    int order_id_ = 0;
};
```

### 4.2 Order State Machine

Track every live order through its full lifecycle to prevent phantom orders and double-fills:

```python
from enum import Enum

class OrderState(Enum):
    PENDING_NEW    = "PENDING_NEW"
    NEW            = "NEW"
    PARTIALLY_FILLED = "PARTIALLY_FILLED"
    FILLED         = "FILLED"
    PENDING_CANCEL = "PENDING_CANCEL"
    CANCELLED      = "CANCELLED"
    REJECTED       = "REJECTED"

class Order:
    def __init__(self, cl_ord_id: str, symbol: str, side: str,
                  qty: int, price: float):
        self.cl_ord_id  = cl_ord_id
        self.symbol     = symbol
        self.side       = side
        self.qty        = qty
        self.price      = price
        self.state      = OrderState.PENDING_NEW
        self.cum_qty    = 0
        self.avg_px     = 0.0

    def on_exec_report(self, exec_type: str, last_qty: int,
                        last_px: float, leaves_qty: int):
        if exec_type == '0':    # New
            self.state = OrderState.NEW
        elif exec_type == '1':  # Partial fill
            self._apply_fill(last_qty, last_px)
            self.state = OrderState.PARTIALLY_FILLED
        elif exec_type == '2':  # Fill
            self._apply_fill(last_qty, last_px)
            self.state = OrderState.FILLED
        elif exec_type == '4':  # Cancelled
            self.state = OrderState.CANCELLED
        elif exec_type == '8':  # Rejected
            self.state = OrderState.REJECTED

    def _apply_fill(self, qty: int, px: float):
        total_cost    = self.avg_px * self.cum_qty + px * qty
        self.cum_qty += qty
        self.avg_px   = total_cost / self.cum_qty
```

---

## 5. FIX Performance Considerations

| Approach | Latency | Notes |
|----------|---------|-------|
| Naive tag-value parse | 10-50 µs | `strtok` + `atof` per field |
| Pre-indexed parse | 2-5 µs | Build tag offset index on SOH scan |
| Binary FIX (SBE) | < 1 µs | Fixed offsets, no parse |
| OUCH/ITCH | < 0.5 µs | Native binary, direct cast |

For latency-sensitive paths, avoid FIX and use OUCH (Ch 11). Use FIX for order management, reconciliation, and broker connectivity where latency > 1ms is acceptable.

---

Next Chapter: [Binary Protocols: ITCH/OUCH/SBE](11_binary_protocols.md)
