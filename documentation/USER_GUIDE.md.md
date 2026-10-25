# RMAPForge Pro User Guide & Developer Manual

## 1\. Quickstart Guide

1. Launch `index.html` in your browser.  
2. In the "Spacecraft Memory Access Config" panel, set the Target Logical Address (e.g. `0xFE`).  
3. Set the Authorization Key (e.g. `0xD2`) and 32-bit Memory Address (e.g. `0x40000000`).  
4. Enter your payload bytes in hex or plain text.  
5. Choose verified write and incrementing address options.  
6. Click **Issue Write Command** to assemble the RMAP frame, compute both Header CRC-8 and Data CRC-8, and inspect the packet hex dump.  
7. Click **Issue Read Command** to generate a corresponding RMAP read request frame.  
8. Click **Run Normative Audit** to evaluate all 10 ECSS-E-ST-50-52C protocol compliance checks.

## 2\. Standalone Headless Python CLI

Run automated regression testing and compliance verification:

python3 rmap-cli.py \--audit \--target 0xFE \--addr 0x40000000