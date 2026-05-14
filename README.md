dump-analyzer
A C++ command-line tool for parsing, analyzing, and GDPR-compliant redaction of system dump files and diagnostic logs from enterprise server platforms.

Problem
Enterprise server dumps contain critical diagnostic data — but they also contain customer PII: hostnames, IP addresses, email addresses, serial numbers, and hardware identifiers. Before a dump can be shared with support teams or analyzed externally, this data must be identified and redacted. Manually doing this is slow, error-prone, and doesn't scale.
dump-analyzer automates this pipeline:

Parse raw dump/log files
Detect and redact PII patterns
Flag anomalous memory and error signatures
Output a clean, analysis-ready file


Features

PII Detection & Redaction — regex-based detection of emails, IPv4/IPv6 addresses, hostnames, hardware serial numbers, and MAC addresses
GDPR-Compliant Output — redacted fields are replaced with consistent tokens (e.g. [REDACTED-EMAIL-001]) so cross-references remain traceable without exposing raw data
Error Pattern Analysis — scans for known hardware/firmware error signatures and memory corruption indicators
Anomaly Flagging — highlights unusual patterns in memory regions or repeated fault codes
Structured Output — outputs JSON summary of findings alongside redacted dump


Build
Requirements:

GCC 9+ or Clang 10+
CMake 3.15+
C++17

bashgit clone https://github.com/atulpal26/dump-analyzer.git
cd dump-analyzer
mkdir build && cd build
cmake ..
make

Usage
bash# Basic redaction
./dump-analyzer --input system_dump.log --output clean_dump.log

# Full analysis with JSON report
./dump-analyzer --input system_dump.log --output clean_dump.log --report report.json

# Specify redaction rules
./dump-analyzer --input system_dump.log --rules config/rules.yaml --output clean_dump.log

Example
Input:
[2024-01-15 10:32:01] ERROR kernel: Memory fault at 0xFFFF8800
  hostname: prod-server-bangalore-042
  contact: admin@customer-corp.com
  serial: SN-78AX2901K
  IP: 192.168.1.105
  fault_code: 0xDEADBEEF
  stack_trace: [kernel_oops+0x14/0x40]
Output (redacted):
[2024-01-15 10:32:01] ERROR kernel: Memory fault at 0xFFFF8800
  hostname: [REDACTED-HOST-001]
  contact: [REDACTED-EMAIL-001]
  serial: [REDACTED-SERIAL-001]
  IP: [REDACTED-IP-001]
  fault_code: 0xDEADBEEF
  stack_trace: [kernel_oops+0x14/0x40]
JSON Report:
json{
  "summary": {
    "total_lines": 312,
    "redacted_fields": 14,
    "anomalies_flagged": 3,
    "error_signatures": ["MEMORY_FAULT", "KERNEL_OOPS"]
  },
  "anomalies": [
    { "line": 47, "type": "REPEATED_FAULT_CODE", "value": "0xDEADBEEF", "count": 5 },
    { "line": 112, "type": "MEMORY_CORRUPTION_PATTERN", "region": "0xFFFF8800-0xFFFF88FF" }
  ]
}

Project Structure
dump-analyzer/
├── src/
│   ├── main.cpp           # Entry point, CLI argument parsing
│   ├── parser.cpp         # Dump file tokenizer and line processor
│   ├── redactor.cpp       # PII detection and redaction engine
│   ├── analyzer.cpp       # Error pattern and anomaly detection
│   └── reporter.cpp       # JSON report generation
├── include/
│   ├── parser.h
│   ├── redactor.h
│   ├── analyzer.h
│   └── reporter.h
├── config/
│   └── rules.yaml         # Configurable redaction rules
├── tests/
│   ├── test_redactor.cpp
│   └── test_analyzer.cpp
├── sample_dumps/
│   └── sample_system.log  # Synthetic sample data for testing
├── CMakeLists.txt
└── README.md

Design Decisions
Why regex over ML for PII detection?
Dump files follow structured formats from specific platforms. Regex rules are deterministic, auditable, and produce zero false negatives for known patterns — which matters in a compliance context. ML introduces probabilistic errors that are unacceptable when GDPR redaction is a hard requirement.
Why consistent redaction tokens instead of hashing?
Tokens like [REDACTED-IP-001] let analysts trace cross-references across a dump (the same IP appearing in multiple events) without exposing the raw value. Hashing achieves privacy but breaks traceability.
Why C++17?
std::filesystem, structured bindings, and std::string_view make the parser significantly cleaner without introducing dependencies. The target environment (enterprise Linux) reliably supports GCC 9+.

Background
Built from experience analyzing system and PHYP dumps on IBM Power enterprise server platforms. The redaction challenge is real — field dumps from customer environments must be sanitized before analysis, and doing this manually at scale is a bottleneck in support workflows.
