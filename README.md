# Ach
rdfi_routing,rdfi_account,amount_cents,individual_name
123456789,987654321,10000,John Doe
021000021,123450987,2500,Jane Smith#!/usr/bin/env python3
import csv
from datetime import datetime

INPUT_CSV = "payments.csv"
OUTPUT_ACH = "output.ach"

# Static config – CHANGE THESE FOR YOUR ORIGINATOR/BANK
IMMEDIATE_DESTINATION = "123456789"   # Bank routing (including leading 0 if any)
IMMEDIATE_ORIGIN = "987654321"       # Your routing or ODFI-assigned ID
DESTINATION_NAME = "DEST BANK"
ORIGIN_NAME = "MY COMPANY"
COMPANY_ID = "9876543210"            # 10-char Company ID (often EIN)
COMPANY_NAME = "MY COMPANY"
ENTRY_DESCRIPTION = "PAYROLL"        # 10-char description
SERVICE_CLASS_CODE = "200"           # 200 = mixed debits/credits
STANDARD_ENTRY_CLASS = "PPD"         # or CCD, etc.
COMPANY_ENTRY_DESC_DATE = ""         # optional
ORIGIN_DFI = IMMEDIATE_ORIGIN[:8]    # first 8 of your routing

def pad(s, length, align="left", filler=" "):
    s = str(s)
    if len(s) > length:
        return s[:length]
    if align == "left":
        return s.ljust(length, filler)
    else:
        return s.rjust(length, filler)

def numeric(n, length):
    return pad(str(int(n)), length, align="right", filler="0")

def file_header_record():
    now = datetime.now()
    file_date = now.strftime("%y%m%d")
    file_time = now.strftime("%H%M")
    return (
        "1" +                      # Record Type Code
        "01" +                     # Priority Code
        pad(IMMEDIATE_DESTINATION, 10, "right", " ") +
        pad(IMMEDIATE_ORIGIN, 10, "right", " ") +
        file_date +                # File Creation Date (YYMMDD)
        file_time +                # File Creation Time (HHMM)
        "A" +                      # File ID Modifier
        "094" +                    # Record Size
        "10" +                     # Blocking Factor
        "1" +                      # Format Code
        pad(DESTINATION_NAME, 23) +
        pad(ORIGIN_NAME, 23) +
        pad("", 8)                 # Reference Code
    )

def batch_header_record(batch_number=1):
    now = datetime.now()
    effective_date = now.strftime("%y%m%d")
    return (
        "5" +                          # Record Type Code
        SERVICE_CLASS_CODE +           # Service Class Code
        pad(COMPANY_NAME, 16) +
        pad(COMPANY_ID, 10) +
        pad("", 20) +                  # Company Descriptive Date
        pad(ENTRY_DESCRIPTION, 10) +
        pad(COMPANY_ENTRY_DESC_DATE, 6) +
        effective_date +               # Effective Entry Date
        "   " +                        # Settlement Date (Julian, left blank)
        "1" +                          # Originator Status Code
        pad(ORIGIN_DFI, 8) +
        numeric(batch_number, 7)
    )

def entry_detail_record(routing, account, amount_cents, name, trace_seq):
    # Simple example: credit to checking (code 22)
    transaction_code = "22"           # 22 = credit checking [web:1]
    rdfi_id = routing[:8]
    check_digit = routing[8]
    return (
        "6" +
        transaction_code +
        rdfi_id +
        check_digit +
        pad(account, 17) +
        numeric(amount_cents, 10) +
        pad("", 15) +                 # Individual ID Number
        pad(name, 22) +
        "  " +                        # Discretionary Data
        "0" +                         # Addenda Record Indicator
        ORIGIN_DFI +
        numeric(trace_seq, 7)
    )

def batch_control_record(entry_count, entry_hash, total_debits, total_credits, batch_number=1):
    return (
        "8" +
        SERVICE_CLASS_CODE +
        numeric(entry_count, 6) +
        numeric(entry_hash, 10) +
        numeric(total_debits, 12) +
        numeric(total_credits, 12) +
        pad(COMPANY_ID, 10) +
        pad("", 19) +              # Message Auth Code
        pad("", 6) +               # Reserved
        pad(ORIGIN_DFI, 8) +
        numeric(batch_number, 7)
    )

def file_control_record(batch_count, block_count, entry_count, entry_hash, total_debits, total_credits):
    return (
        "9" +
        numeric(batch_count, 6) +
        numeric(block_count, 6) +
        numeric(entry_count, 8) +
        numeric(entry_hash, 10) +
        numeric(total_debits, 12) +
        numeric(total_credits, 12) +
        pad("", 39)                # Reserved
    )

def round_up_blocks(record_count):
    # 10 records per block, 94 chars each [web:6][web:9]
    blocks = (record_count + 9) // 10
    return blocks

def main():
    entries = []
    entry_hash = 0
    total_credits = 0
    total_debits = 0  # not used in this simple credit-only example

    with open(INPUT_CSV, newline="") as f:
        reader = csv.DictReader(f)
        seq = 1
        for row in reader:
            routing = row["rdfi_routing"].strip()
            account = row["rdfi_account"].strip()
            amount_cents = int(row["amount_cents"])
            name = row["individual_name"][:22]

            entries.append(entry_detail_record(routing, account, amount_cents, name, seq))

            entry_hash += int(routing[:8])
            total_credits += amount_cents
            seq += 1

    entry_count = len(entries)
    # Entry hash is truncated to 10 rightmost digits [web:3][web:6]
    entry_hash = int(str(entry_hash)[-10:])

    records = []
    records.append(file_header_record())
    records.append(batch_header_record())

    records.extend(entries)

    records.append(batch_control_record(
        entry_count=entry_count,
        entry_hash=entry_hash,
        total_debits=total_debits,
        total_credits=total_credits
    ))

    # Count records so far to compute blocks and file control
    record_count = len(records) + 1  # +1 for file control
    block_count = round_up_blocks(record_count)

    records.append(file_control_record(
        batch_count=1,
        block_count=block_count,
        entry_count=entry_count,
        entry_hash=entry_hash,
        total_debits=total_debits,
        total_credits=total_credits
    ))

    # Pad with 9-records to fill last block (if needed) [web:6][web:9]
    while len(records) % 10 != 0:
        records.append("9" + "9" * 93)

    with open(OUTPUT_ACH, "w", newline="") as f:
        for r in records:
            f.write(r[:94].ljust(94) + "
")

    print(f"Wrote {len(records)} records to {OUTPUT_ACH}")

if __name__ == "__main__":
    main()
    chmod +x csv_to_ach.py
python3 csv_to_ach.py
def build_ach_from_csv(input_csv):
    entries = []
    entry_hash = 0
    total_credits = 0
    total_debits = 0

    with open(input_csv, newline="") as f:
        reader = csv.DictReader(f)
        seq = 1
        for row in reader:
            routing = row["rdfi_routing"].strip()
            account = row["rdfi_account"].strip()
            amount_cents = int(row["amount_cents"])
            name = row["individual_name"][:22]

            entries.append(entry_detail_record(routing, account, amount_cents, name, seq))
            entry_hash += int(routing[:8])
            total_credits += amount_cents
            seq += 1

    entry_count = len(entries)
    entry_hash = int(str(entry_hash)[-10:])

    records = []
    records.append(file_header_record())
    records.append(batch_header_record())
    records.extend(entries)
    records.append(batch_control_record(
        entry_count=entry_count,
        entry_hash=entry_hash,
        total_debits=total_debits,
        total_credits=total_credits
    ))

    record_count = len(records) + 1
    block_count = round_up_blocks(record_count)

    records.append(file_control_record(
        batch_count=1,
        block_count=block_count,
        entry_count=entry_count,
        entry_hash=entry_hash,
        total_debits=total_debits,
        total_credits=total_credits
    ))

    while len(records) % 10 != 0:
        records.append("9" + "9" * 93)

    # Return full ACH file contents as a single string
    return "".join(r[:94].ljust(94) + "
" for r in records)

def main():
    ach_text = build_ach_from_csv(INPUT_CSV)
    with open(OUTPUT_ACH, "w", newline="") as f:
        f.write(ach_text)
    print(f"Wrote ACH file to {OUTPUT_ACH}")
    return OUTPUT_ACH  # <- return value
    import requests

API_KEY = "YOUR_FIXER_API_KEY"  # Replace with your Fixer API key
BASE_URL = "http://data.fixer.io/api/latest"

def convert_usd_to_currencies(amount_usd, target_currencies):
    """
    Convert a USD amount to multiple target currencies using Fixer.io API.
    
    Args:
        amount_usd (float): The ACH amount in USD to convert.
        target_currencies (list): List of currency codes to convert to.
        
    Returns:
        dict: Mapping of currency code to converted amount, rounded to 4 decimals.
    Raises:
        ValueError: If API call fails or returns error.
    """
    # Fixer free API tier has EUR as base currency, so we fetch USD and target rates relative to EUR.
    params = {
        'access_key': API_KEY,
        'symbols': ','.join(target_currencies) + ',USD'
    }

    response = requests.get(BASE_URL, params=params)
    data = response.json()

    if not data.get('success', False):
        raise ValueError(f"API error: {data.get('error', {}).get('info', 'Unknown error')}")

    rates = data['rates']
    
    usd_rate = rates['USD']
    conversions = {}

    for curr in target_currencies:
        target_rate = rates[curr]
        # Convert USD->EUR->Target: amount in USD / USD per EUR * target per EUR
        converted_amount = (amount_usd / usd_rate) * target_rate
        conversions[curr] = round(converted_amount, 4)

    return conversions

# Example usage
if __name__ == "__main__":
    try:
        usd_amount = 1000.00
        currencies = ['EUR', 'JPY', 'GBP', 'CAD']
        conversion_results = convert_usd_to_currencies(usd_amount, currencies)
        # Returns dict you can store or use in your accounting entries
        print(conversion_results)
    except Exception as e:
        print(f"Conversion failed: {e}")
