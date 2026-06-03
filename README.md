# IoT Challenge #3 — Node-RED + LoRaWAN

Third challenge for the Internet of Things course at Politecnico di Milano (ANTLab), academic year 2025/2026.

The assignment had two parts: a Node-RED flow that processes a ZigBee capture and routes data over MQTT + ThingSpeak (Part 1), and a short theoretical exercise on LoRaWAN spreading factors (Part 2). I built the Node-RED flow; the LoRaWAN exercise was done by my teammate Anasuya. More on the split at the bottom.

## What the flow does

Everything lives on a single Node-RED tab, organized in four branches that run in parallel:

- **Initialization** — on deploy, reads `challenge3.csv` into memory and indexes it by Packet Number so lookups are O(1) instead of scanning ~5200 rows every time. Also rewrites the headers of the two append-mode CSV files so a re-deploy doesn't pile up old data.
- **Publisher** — every second, generates a random ID in `[0, 30000]` plus a UNIX timestamp, publishes it as JSON to `challenge3/id_generator` on the local Mosquitto broker (`localhost:1884`), and logs it to `id_log.csv`.
- **Subscriber / processing** — subscribes to the same topic. For each message it: (1) counts it (the 200-message limit gate sits here, so even ignored packets count, as the spec requires); (2) computes `N = ID mod 5218`; (3) fetches the matching row; (4) classifies it as `zbee_zcl` / `link_status` / `ignore` and routes it.
  - **ZBEE_ZCL path**: builds the MQTT payload the assignment asks for, publishes it (rate-limited to 10/min), and in parallel extracts only the *RMS Current / RMS Voltage / Active Power* attributes from valid response packets. The values feed `filtered_elems.csv` and two live charts on the dashboard.
  - **Link Status path**: keeps an ordered map of (source, destination → cost) and updates it as new Link Status packets arrive — keeping the original arrival order of each link.
- **Finalization** — fires exactly when the 200th message lands. Writes `outgoing_cost.csv` in arrival order, then picks the smallest source address (treating addresses as hex), sorts its destinations ascending, and pushes the costs to ThingSpeak (field1) at one per 20 seconds — the free-tier limit.

A full node-by-node walkthrough is in `Challenge.pdf`.

## Bits that took me longer than expected

A few things I had to actually think about, in case it helps anyone reading the flow:

- The Command String column in `challenge3.csv` is **Python-style pseudo-JSON**: single quotes, `True`/`False`/`None`. Before parsing I rewrite it to real JSON (double quotes, `true`/`false`/`null`).
- Filtering ZBEE_ZCL packets is tricky because **not all of them carry the data**. Only *Read Attributes Response* packets have Attribute + Status + Data Type all populated together. *Read Attributes* requests, *Report Attributes*, and *Default Response* fall through the same column structure but are missing one piece each, so my filter rejects them at the gate.
- Int16 and Uint16 values come as **two separate lists** in the Command String. You can't just zip Attribute and one value list together — you need a cursor per data type that advances through each list independently while you scan the attributes.
- The 200-message gate sits **before** the classification step, otherwise dropped/ignored messages don't count and you overshoot. The spec is explicit about this.

## How to run it

You need:

- Node-RED (tested on 4.x)
- Mosquitto listening on `localhost:1884` (note: NOT the default 1883, you have to set the port in the config)
- The standard Node-RED palette + `node-red-dashboard` for the charts
- `challenge3.csv` in Node-RED's working directory

Steps:

1. In Node-RED, top-right menu → *Import* → paste the contents of `nodered.txt`.
2. The file nodes use relative paths (`./challenge3.csv` and `./challenge3_output/*`). Create a `challenge3_output/` folder in Node-RED's working dir before the first deploy, or change the paths to whatever you want.
3. Open the function node *Split into individual ThingSpeak msg* and replace `YOUR_THINGSPEAK_WRITE_API_KEY` with your own ThingSpeak write key.
4. Deploy. The CSV gets loaded automatically and the headers of the two append-mode files are rewritten. The publisher starts emitting straight away and self-stops after 200 IDs.

## Part 2 — LoRaWAN exercise

The second part was a pen-and-paper exercise on a European LoRaWAN network (40 nodes, λ = 2 pkt/min, 20-byte payload, BW 125 kHz). **My teammate Anasuya did this part** — her work and write-up are in `Exercise.pdf`. Quick summary:

- **EQ1**: under pure ALOHA, **SF7** is the largest spreading factor that keeps packet success rate ≥ 75% (≈ 82.5%). Already SF8 drops to ~70%.
- **EQ2**: when performance after deployment is non-uniform across nodes, that's a link-budget issue, not a collision issue. The right action is **moving the nodes closer to the gateway**, not changing the SF or reducing the node count.

Full reasoning, tables and computations in the PDF.

## Repo layout

```
.
├── nodered.txt              # Node-RED flow export, import in Node-RED
├── challenge3.csv           # input dataset (provided by the course)
├── id_log.csv               # log of the 200 published IDs
├── filtered_elems.csv       # filtered ZBEE_ZCL rows
├── outgoing_cost.csv        # ZigBee link cost map
├── Challenge.pdf            # Part 1 report — flow explained node by node
├── Exercise.pdf             # Part 2 report — LoRaWAN exercise
└── nodes_flow_photo.png     # screenshot of the full flow
```

## Notes

- The ThingSpeak write API key has been replaced with `YOUR_THINGSPEAK_WRITE_API_KEY`. You need your own to actually push to a channel.
- The ThingSpeak channel I used (ID `3340710`) is public, so anyone can view the 8 data points I sent during the assignment.
- The Mosquitto broker has to listen on port `1884`, not the default `1883`, otherwise the MQTT in/out nodes stay disconnected.
- File node paths are relative — they resolve to wherever Node-RED is started from, not to the location of `nodered.txt`.

## Team & credits

This was a pair project:

- **Golsa Shokri** — Part 1 (Node-RED flow: MQTT routing, ZigBee packet parsing, CSV outputs, dashboard charts, ThingSpeak integration), report writing for Part 1.
- **Anasuya Satapathy** — Part 2 (LoRaWAN spreading factor analysis, EQ1 and EQ2 derivations), report writing for Part 2.

Politecnico di Milano — Internet of Things course, May 2026.
