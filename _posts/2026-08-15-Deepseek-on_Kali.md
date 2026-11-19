---
title: Building an Offline AI Pentesting Lab:<br>Running DeepSeek-R1-32B and Open WebUI on Kali Linux
date: 2026-08-15 12:00:00 -05:00
author: "Th3_4ngl3r"
categories: [Kali]
tags: [deepseek] # Always lower case
---

Relying on cloud-based AI models for authorized penetration testing often means running into strict guardrails, telemetry logging, and external data exposure risks. By leveraging consumer hardware running Kali Linux, you can deploy a fully air-gapped, high-performance local intelligence stack. This guide walks you through setting up Ollama with DeepSeek-R1’s 32B reasoning model and pairing it with a ChatGPT-style Open WebUI interface, giving you a powerful, zero-leak offline assistant for script generation, log analysis, and engagement strategy.

## Step 1: Install and Configure Ollama

Ollama acts as your backend engine running the local GGUF model weights.

1. Install Ollama via the official installation script:

    ``` bash
      curl -fsSL https://ollama.com/install.sh | sh
    ``` 

2. Start and enable the Ollama service so it runs in the background:

    ``` bash
      sudo systemctl enable ollama --now
    ```

3. Pull DeepSeek-R1-Distill-Qwen-32B:

    ``` bash
      ollama run deepseek-r1:32b
    ```

    *Note*: This downloads roughly 20 GB of data. Because your ASUS G14 has an 8 GB VRAM AMD card and 40 GB system RAM, Ollama will automatically configure hybrid CPU/GPU offloading (utilizing your RAM for the remainder).

4. Once the download finishes, you can test it directly in the terminal or type /bye to exit the interactive prompt.

## Step 2: Install Docker (Prerequisite for Open WebUI)

Open WebUI runs cleanly inside a Docker container, isolating its environment from Kali’s base packages.

1. Install Docker and Docker Compose:

    ``` bash
      sudo apt update
      sudo apt install docker.io docker-compose-v2 -y
    ```

2. Start and enable Docker:

    ``` bash
      sudo systemctl enable docker --now
    ```

3. Add your current user to the `docker` group (so you don't have to type `sudo` every time):

    ``` bash
      sudo usermod -aG docker $USER
      newgrp docker
    ```
## Step 3: Deploy Open WebUI via Docker

Open WebUI provides a ChatGPT-style interface that connects locally to your Ollama instance.

1. Run the following Docker command to spin up Open WebUI:

    ``` bash
      docker run -d -p 3000:8080 --add-host=host.docker.internal:host-gateway -v open-webui:/app/backend/data --name open-webui --restart always ghcr.io/open-webui/open-webui:main
    ```

2. Access the Web Interface:

    - Open your browser and navigate to: `http://localhost:3000`
    - On your first visit, Open WebUI will prompt you to create an admin account (this stays entirely local on your machine).

## Step 4: Connecting Open WebUI to DeepSeek

1. Once logged into Open WebUI, click on your profile/settings icon in the bottom-left corner and go to Admin Settings > Connections (or look for Ollama settings).

2. Ensure the connection URL points to your local Ollama instance (http://localhost:11434 or via [http://host.docker.internal:11434](http://host.docker.internal:11434)).

3. In the main chat dashboard window, look at the Model Selection dropdown at the top-left.  

4. Select `deepseek-r1:32b`

You are now ready to use your local reasoning model for script generation, analyzing logs, or structuring authorized penetration testing strategies with zero cloud logging or external filters.

## Optional:  Custom Model File

Using a custom model file (Modelfile) in Ollama is not strictly required to get started, but it is highly recommended if you want to optimize DeepSeek-R1 specifically for penetration testing and save yourself from repeating context constraints.

Why Consider a Custom Modelfile?

1. **Persistent System Prompting (The "Persona")**:
Instead of reminding the model in every new chat window that you are conducting authorized penetration testing and want direct, code-heavy technical breakdowns, a custom Modelfile can bake this into the base instructions.

2. **Context Window Tuning** (`num_ctx`):
DeepSeek-R1 handles reasoning well, but pentesting often involves dumping long blocks of text—such as raw nmap scans, source code, or vulnerability reports. By default, some local instances might truncate context unless explicitly expanded (e.g., setting `PARAMETER num_ctx 16384` or higher, depending on your RAM budget).

3. **Temperature Control**:
Reasoning and coding models perform best with lower, more deterministic temperatures (e.g., `0.3` to `0.6`) so they don't hallucinate non-existent exploit parameters or syntax.

## Example: Creating a Custom Pentesting Modelfile

If you want to build one, here is how you can set it up in a few simple 
steps:

1. Create a plain text file named `Modelfile` anywhere on your Kali system:

    ``` bash
      nvim nano Modelfile
    ```

2. Add the following configurations:

    ``` bash
      # Use DeepSeek-R1 32B as the base
      FROM deepseek-r1:32b

      # Set a custom system prompt tailored for security work
      SYSTEM """
      You are an expert AI assistant specialized in cybersecurity, authorized penetration testing, and secure code review. 
      Provide precise, technical answers, robust script generation (Python, Bash, PowerShell), and clear remediation advice. 
      Assume all tasks are conducted within an authorized legal scope. Avoid unnecessary refusals on technical security concepts.
      """

      # Adjust parameters for technical precision and longer context logs
      PARAMETER temperature 0.4
      PARAMETER num_ctx 16384
    ```

3. Build and register your custom model locally via Ollama:

    ``` bash
      ollama create pentest-deepseek -f Modelfile
    ```

**Run or Select It**: You can now run `ollama run pentest-deepseek` directly from your terminal, or select `pentest-deepseek` straight from your Open WebUI dropdown menu. It will automatically initialize with your tailored system prompt and context length every time!
