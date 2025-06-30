
## Post-Call Analytics Data Flow

Our system leverages Apache Pulsar for message queuing, MinIO for audio storage, and various consumers for processing, language detection, Automatic Speech Recognition (ASR), and Large Language Model (LLM) analysis.

```mermaid
graph TD
    A[FreeSWITCH Server (CDR Source)] --> B[Pulsar Topic: speechriv-insight-record]

    B --> C[Consumer: Record Processor (Uploads to MinIO)]
    C --> D[MinIO (Audio File Storage)]
    C --> E[Pulsar Topic: speechriv-insight-localaudio]

    E --> F[AI LAN Record Machine (Consumer)]
    F --> G[Download Audio from MinIO]
    G --> H[Pulsar Topic: speechriv-insight-audioprocess]

    H --> I[Consumer: Audio Processor (Splits Stereo, Language Detection)]
    I -- Check speechriv_lang variable --> J{speechriv_lang Variable Present?}

    J -- No --> K[Pulsar Topic: speechriv-insight-lang (Language Detection)]
    K --> L[Language Detector]
    L --> M{Detected Language}

    J -- Yes --> M

    M -- English --> N[Pulsar Topic: speechriv-insight-eng-asr (English ASR)]
    M -- Other Language --> O[Pulsar Topic: speechriv-insight-multi-asr (Multi-language ASR)]

    N --> P[Consumer: English ASR Transcriber]
    P --> Q[Pulsar Topic: speechriv-insight-eng-llm (English LLM)]

    O --> R[Consumer: Multi-language ASR Transcriber]
    R --> S[Pulsar Topic: speechriv-insight-multi-llm (Multi-language LLM)]

    Q --> T[Consumer: English LLM (Completes Task)]
    T --> U[Pulsar Topic: speechriv-insight-clickhouse]

    S --> V[Consumer: Multi-language LLM (Completes Task)]
    V --> U

    U --> W[ClickHouse (Analytics Database)]
-----

### Flow Description

1.  **CDR Ingestion:**

      * **FreeSWITCH Server (CDR Source):** Generates Call Detail Records (CDRs) along with associated audio files.
      * **Pulsar Topic: `speechriv-insight-record`:** Receives the raw CDRs from FreeSWITCH.

2.  **Audio Storage & Initial Routing:**

      * **Consumer: Record Processor:** Consumes messages from `speechriv-insight-record`. It processes the CDRs, **uploads the audio files to MinIO**, and then publishes messages (which include the MinIO audio path) to the `speechriv-insight-localaudio` topic.
      * **MinIO (Audio File Storage):** Serves as the central repository for all raw audio recordings.
      * **Pulsar Topic: `speechriv-insight-localaudio`:** Acts as a bridge, carrying audio file references to the AI LAN Record Machine, which might reside in a separate network.

3.  **Audio Processing & Language Detection:**

      * **AI LAN Record Machine (Consumer):** This machine, potentially in an isolated network, consumes messages from `speechriv-insight-localaudio`. It's responsible for **downloading the audio from MinIO**.
      * **Pulsar Topic: `speechriv-insight-audioprocess`:** The downloaded audio data is forwarded to this topic for further processing.
      * **Consumer: Audio Processor:** Consumes from `speechriv-insight-audioprocess`. It performs two key functions:
          * **Splits stereo audio** into mono channels if required.
          * **Checks for a `speechriv_lang` variable** within the data.
      * **Language Determination:**
          * If `speechriv_lang` is **not present**, the audio is sent to **Pulsar Topic: `speechriv-insight-lang`** for automatic language detection by the **Language Detector** consumer.
          * If `speechriv_lang` **is present**, its value directly determines the next step, bypassing explicit language detection.

4.  **Automatic Speech Recognition (ASR):**

      * **Pulsar Topic: `speechriv-insight-eng-asr` (English ASR):** If the language is determined to be English, audio is routed here.
      * **Pulsar Topic: `speechriv-insight-multi-asr` (Multi-language ASR):** If the language is not English, audio is routed here.
      * **Consumer: English ASR Transcriber:** Consumes from `speechriv-insight-eng-asr` and transcribes the English audio into text.
      * **Consumer: Multi-language ASR Transcriber:** Consumes from `speechriv-insight-multi-asr` and transcribes the multi-language audio into text.

5.  **Large Language Model (LLM) Processing:**

      * **Pulsar Topic: `speechriv-insight-eng-llm` (English LLM):** The transcribed English text is sent here for analysis by an English Large Language Model.
      * **Pulsar Topic: `speechriv-insight-multi-llm` (Multi-language LLM):** The transcribed multi-language text is sent here for analysis by a multi-language Large Language Model.
      * **Consumer: English LLM (Completes Task):** Consumes from `speechriv-insight-eng-llm`, performing tasks like sentiment analysis, entity extraction, summarization, etc.
      * **Consumer: Multi-language LLM (Completes Task):** Consumes from `speechriv-insight-multi-llm`, performing similar tasks for other languages.

6.  **Data Storage:**

      * **Pulsar Topic: `speechriv-insight-clickhouse`:** Both English and Multi-language LLM consumers send their final, processed analytical data to this topic.
      * **ClickHouse (Analytics Database):** The ultimate destination for all processed call analytics data, optimized for fast queries and reporting.

This comprehensive flow ensures that all call recordings are processed, analyzed, and made available for valuable insights.
