# Binary Protocols: ITCH, OUCH, and SBE

Exchange protocols define the wire format for market data consumption and order entry. High-frequency systems require zero-copy, zero-parse binary protocols. This chapter covers Nasdaq ITCH (market data), OUCH (order entry), and the Simple Binary Encoding (SBE) standard used by CME and others.

---

## 1. Protocol Taxonomy

| Protocol | Direction | Transport | Format | Typical Latency |
|----------|-----------|-----------|--------|-----------------|
| FIX 4.x  | Both      | TCP       | Text tag-value | 50–500 µs |
| ITCH 5.0 | Feed → Client | UDP Multicast | Binary fixed-length | 1–10 µs |
| OUCH 4.x | Client → Exchange | TCP | Binary fixed-length | 1–5 µs |
| SBE      | Both      | TCP/UDP   | Binary, schema-defined | 1–10 µs |

---

## 2. Nasdaq ITCH 5.0 (Market Data Feed)

ITCH is a one-way binary data feed. The exchange multicasts order book events (add, cancel, execute, delete) as a stream of fixed-length messages over UDP. Because messages have fixed lengths and field positions, the receiver casts the raw byte buffer directly to a C++ struct — zero parsing overhead.

### 2.1 Wire Format

Each ITCH message begins with a 2-byte length prefix followed by a 1-byte message type. Fields are big-endian.

```
┌─────────────┬──────────┬──────────────────────────┐
│ Length (2B) │ Type (1B)│ Payload (variable)        │
└─────────────┴──────────┴──────────────────────────┘
```

### 2.2 Key Message Types

| Type Byte | Message | Description |
|-----------|---------|-------------|
| `'A'` | Add Order | New resting limit order |
| `'F'` | Add Order (MPID) | Add with market participant ID |
| `'E'` | Order Executed | Fill against existing order |
| `'C'` | Order Executed w/ Price | Fill at off-book price |
| `'X'` | Order Cancel | Partial cancel |
| `'D'` | Order Delete | Full cancel |
| `'U'` | Order Replace | Cancel + re-add |
| `'P'` | Trade (non-cross) | Matched trade |

### 2.3 C++ Struct Binding

```cpp
#pragma pack(push, 1)

struct ITCHAddOrder {
    char     msg_type;               // 'A'
    uint16_t stock_locate;
    uint16_t tracking_number;
    uint64_t timestamp_ns;           // nanoseconds since midnight
    uint64_t order_ref;
    char     side;                   // 'B' = buy, 'S' = sell
    uint32_t shares;
    char     stock[8];               // right-padded with spaces
    uint32_t price;                  // integer, divide by 10000 for dollars
};

struct ITCHOrderExecuted {
    char     msg_type;               // 'E'
    uint16_t stock_locate;
    uint16_t tracking_number;
    uint64_t timestamp_ns;
    uint64_t order_ref;
    uint32_t executed_shares;
    uint64_t match_number;
};

#pragma pack(pop)
```

### 2.4 Feed Handler Implementation

```cpp
void FeedHandler::on_packet(const uint8_t* buf, size_t len) {
    size_t offset = 0;
    while (offset < len) {
        uint16_t msg_len = ntohs(*reinterpret_cast<const uint16_t*>(buf + offset));
        offset += 2;
        char msg_type = buf[offset];

        switch (msg_type) {
            case 'A': {
                auto* msg = reinterpret_cast<const ITCHAddOrder*>(buf + offset);
                book_builder_.on_add(
                    be64toh(msg->order_ref),
                    msg->side,
                    be32toh(msg->shares),
                    be32toh(msg->price)
                );
                break;
            }
            case 'E': {
                auto* msg = reinterpret_cast<const ITCHOrderExecuted*>(buf + offset);
                book_builder_.on_execute(
                    be64toh(msg->order_ref),
                    be32toh(msg->executed_shares)
                );
                break;
            }
            case 'D': {
                auto* msg = reinterpret_cast<const ITCHOrderDelete*>(buf + offset);
                book_builder_.on_delete(be64toh(msg->order_ref));
                break;
            }
        }
        offset += msg_len;
    }
}
```

### 2.5 Line Arbitration

Nasdaq sends identical feeds on two UDP multicast groups (Feed A and Feed B) for redundancy. A robust feed handler must:
1. Maintain a **sequence number** per feed.
2. Accept whichever packet arrives first, discard duplicates.
3. Detect gaps (sequence number jump) and request retransmission via **GapFill** (a separate TCP channel).

```cpp
void FeedHandler::on_udp_packet(int feed_id, uint64_t seq, const uint8_t* buf) {
    if (seq <= last_seq_) return;     // duplicate from other feed
    if (seq > last_seq_ + 1) {
        request_gap_fill(last_seq_ + 1, seq - 1);
        return;
    }
    last_seq_ = seq;
    on_packet(buf, len_);
}
```

---

## 3. Nasdaq OUCH 4.x (Order Entry)

OUCH is the counterpart for sending orders to Nasdaq. All messages are fixed-length binary over a persistent TCP connection. Because fields are fixed-position, the exchange firmware can parse an order in a single DMA read.

### 3.1 Enter Order Message

```cpp
#pragma pack(push, 1)

struct OUCHEnterOrder {
    char     packet_type = 'O';          // 'O' = Enter Order
    char     order_token[14];            // client-assigned unique ID
    char     buy_sell = 'B';             // 'B' or 'S'
    uint32_t shares;                     // big-endian
    char     stock[8];                   // right-padded
    uint32_t price;                      // integer, /10000
    uint32_t time_in_force;              // 0 = IOC, 99999 = day
    char     firm[4];
    char     display = 'Y';              // 'Y' = displayed, 'N' = hidden
    char     capacity = 'O';             // 'O' = other, 'P' = principal
    char     intermarket_sweep = 'N';
    uint32_t min_qty = 0;
    char     cross_type = 'N';
    char     customer_type = ' ';
};

#pragma pack(pop)
```

### 3.2 Acknowledgment Flow

After sending `OUCHEnterOrder`, the exchange responds with:
- **Order Accepted** (`'A'`): Contains an exchange-assigned `order_reference_number`.
- **Order Rejected** (`'J'`): Contains a rejection reason code.
- **Order Executed** (`'E'`): Contains fill details.
- **Order Canceled** (`'C'`): Confirms cancel.

Track pending orders by `order_token` (client side) until exchange assigns `order_reference_number`. All cancel/replace messages reference the exchange's number.

---

## 4. Simple Binary Encoding (SBE)

SBE is a FIX-standard binary encoding used by CME Group's MDP 3.0 feed and iLink 3 order entry. Unlike ITCH (exchange-specific), SBE is a general encoding schema: message layouts are defined in XML and code-generated.

### 4.1 Design Principles

- **Fixed-length fields at fixed offsets** — no length prefixes within fields.
- **Little-endian** (CME convention; contrast with ITCH big-endian).
- **Template ID** in each message header identifies message type without field scanning.
- **Repeating groups** for multi-leg or multi-instrument messages — length-prefixed group header, fixed-width rows.

### 4.2 SBE Message Header

```cpp
struct SBEHeader {
    uint16_t block_length;    // length of root block (no repeating groups)
    uint16_t template_id;     // message type (e.g., 50 = MDIncrementalRefreshBook)
    uint16_t schema_id;
    uint16_t version;
};
```

### 4.3 CME MDP 3.0 Incremental Refresh

```cpp
struct MDIncrementalRefreshBook50 {
    SBEHeader    header;
    int64_t      transact_time;       // nanoseconds since epoch
    uint8_t      match_event_indicator;
    // Repeating group: MDEntries
    struct MDEntry {
        int64_t  md_entry_px;         // price, mantissa only; exponent from schema
        int32_t  md_entry_size;
        int32_t  security_id;
        uint32_t rpt_seq;
        int32_t  number_of_orders;
        uint8_t  md_update_action;    // 0=New, 1=Change, 2=Delete
        char     md_entry_type;       // '0'=Bid, '1'=Ask, '2'=Trade
    };
};
```

### 4.4 Code Generation

CME provides SBE XML schema files (`.xml`). Use the reference SBE tool or `real-logic/simple-binary-encoding` to generate typed C++ decoders:

```bash
java -jar sbe-tool.jar \
  --sbe-schema=CME-MDP3-schema.xml \
  --output-dir=generated/ \
  --target-namespace=cme
```

Generated decoders handle field offset arithmetic, repeating group iteration, and null value checks. Use generated code on all non-critical paths; hand-roll only the hottest message types.

---

## 5. Sequencing and Timestamp Discipline

Exchange timestamps are nanosecond-resolution. Local timestamps (for latency measurement) require hardware timestamping:

```cpp
// Software timestamp — subject to OS scheduling jitter
struct timespec ts;
clock_gettime(CLOCK_REALTIME, &ts);
uint64_t ns = ts.tv_sec * 1'000'000'000ULL + ts.tv_nsec;

// Hardware timestamp (Solarflare / exaNIC) via SO_TIMESTAMPING
// Configured at socket level; arrives as ancillary cmsg data
```

**Latency measurement convention:**
- `T1`: exchange-stamped send time (in ITCH `timestamp_ns`)
- `T2`: NIC hardware receive timestamp (via `SO_TIMESTAMPING`)
- `T3`: software receive timestamp (after OS interrupt)
- `T4`: order sent timestamp

Wire latency = T2 − T1. Processing latency = T4 − T2.

---

Next Chapter: [Network Optimization & Co-location](12_network_optimization.md)
