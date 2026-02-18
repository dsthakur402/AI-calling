# EC2 Technical Architecture & System Report

**Project:** Real-Time Conversational AI Voice Server\
**Instance Role:** GPU-based AI Inference Server

------------------------------------------------------------------------

## 1. Executive Summary

This EC2 instance hosts a real-time conversational AI engine that
processes live audio streams and generates AI voice responses.

It performs the following pipeline in real time:

1.  Speech-to-Text (STT)\
2.  Large Language Model (LLM) processing\
3.  Text-to-Speech (TTS)\
4.  Audio streaming back to caller

The system is GPU-enabled and optimized for low-latency conversational
AI workloads.

------------------------------------------------------------------------

## 2. Infrastructure Overview

### Compute

-   **GPU:** NVIDIA A10G (24GB VRAM)
-   **CUDA Driver Version:** 13.0
-   **GPU Memory Usage:** \~6.5GB (TTS model)
-   **Application Runtime:** Python (asyncio + websockets)

------------------------------------------------------------------------

### Storage

#### Root Volume

-   Total: 97GB\
-   Used: 80GB (82%)\
-   Primary Consumers:
    -   Multiple CUDA installations (\~41GB)
    -   AI model caches (\~7--8GB)
    -   System libraries and dependencies

#### NVMe Ephemeral Volume

-   Total: 229GB\
-   Used: \~0%\
-   Recommended usage:
    -   Model cache relocation
    -   Temporary large files
    -   Future scaling storage

------------------------------------------------------------------------

## 3. Application Architecture

### Project Directory

    /home/ubuntu/conversational-ai/server

### Core Components

  Component              Responsibility
  ---------------------- ---------------------------------------
  main.py                Entry point (WebSocket + HTTP server)
  websocket_handler.py   Manages streaming connections
  stt_processor.py       Speech recognition
  llm_processor.py       AWS Bedrock integration
  tts_processor.py       GPU-based voice generation
  vad_processor.py       Voice Activity Detection
  pipeline.py            Processing orchestration

------------------------------------------------------------------------

## 4. AI Model Stack

### Speech-to-Text (STT)

-   Library: faster-whisper
-   Models available: tiny, base, small
-   Stored locally in model cache
-   Runs on CPU/GPU (lightweight)

### Large Language Model (LLM)

-   Platform: AWS Bedrock
-   Access method: boto3 SDK
-   Model ID defined in configuration
-   Hosted externally (no local LLM weights)

### Text-to-Speech (TTS)

-   Model: Chatterbox Turbo
-   Disk size: \~3.8GB
-   GPU memory usage: \~6.5GB
-   Primary GPU workload

------------------------------------------------------------------------

## 5. Network & Ports

  Port   Purpose
  ------ ------------------------------------
  9000   WebSocket streaming
  9001   HTTP health & metrics
  9100   Node exporter (system metrics)
  9400   NVIDIA DCGM exporter (GPU metrics)

------------------------------------------------------------------------

## 6. End-to-End Call Flow

Phone Call\
→ Twilio SIP\
→ LiveKit\
→ Node Application\
→ WebSocket (Port 9000)\
→ VAD\
→ STT\
→ LLM (AWS Bedrock)\
→ TTS (GPU)\
→ Audio Stream Back\
→ Caller hears AI response

------------------------------------------------------------------------

## 7. Runtime Characteristics

-   Single Python process
-   Async event loop architecture
-   Single GPU context
-   Real-time streaming optimized
-   Shared GPU resources across sessions

------------------------------------------------------------------------

## 8. Observability

Monitoring tools enabled: - Node Exporter (system metrics) - DCGM
Exporter (GPU metrics) - HTTP /health endpoint - HTTP /metrics endpoint

------------------------------------------------------------------------

## 9. Current Limitations

1.  Monolithic single-process design\
2.  Shared GPU context for all sessions\
3.  No horizontal scaling\
4.  Root disk nearing capacity (82%)\
5.  Multiple unused CUDA versions consuming disk space

Estimated safe concurrency: Low-to-moderate (3--10 concurrent sessions
depending on TTS load).

------------------------------------------------------------------------

## 10. Recommendations

### Immediate

-   Remove duplicate model cache from root user
-   Consolidate unused CUDA versions
-   Move model cache to NVMe disk

### Medium-Term

-   Separate TTS into dedicated GPU microservice
-   Introduce multiprocessing worker pool
-   Implement request queue for GPU tasks
-   Enable horizontal scaling with multiple AI instances
-   Configure Auto Scaling Group for resilience

------------------------------------------------------------------------

## 11. System Classification

This EC2 instance functions as:

> A GPU-powered real-time conversational AI inference server designed
> for live voice interactions.

It combines streaming, speech recognition, cloud LLM integration,
GPU-based voice synthesis, and monitoring into a single runtime
environment.
