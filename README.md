# AIOS-Bare-Metal-Architectures
AI Systems Developer | Building Sovereign Local AIOS &amp; Bare-Metal Architectures
#!/usr/bin/env python3
import os
import sys
import re
import psutil
import requests
import subprocess

# System Configuration
DYNA_NAME = "D.Y.N.A."
DYNA_FULL = "Dynamic Yield & Neural Assistant"
OLLAMA_URL = "http://localhost:11434/api/generate"
MODEL_NAME = "qwen3.6:35b-a3b"  

SYSTEM_PROMPT = f"""
You are {DYNA_NAME} ({DYNA_FULL}), a terminal-native system assistant and operating core.
You respond in a direct, tactical, and concise CLI manner.
Assist the operator with system administration, code analysis, and terminal operations.
CRITICAL RULE: Whenever you suggest a terminal command for the operator to run, enclose the exact command in single backticks (e.g., `ls -la`).
Do NOT include place-holders like `<PORT>` or `<PATH>` inside backticks—provide actual ready-to-run examples.
"""

session_command_history = []

def get_system_metrics():
    """Reads system diagnostic parameters directly from OS."""
    cpu = psutil.cpu_percent(interval=0.3)
    ram = psutil.virtual_memory()
    disk = psutil.disk_usage('/')
    return (
        f"CPU Load  : {cpu}%\n"
        f"RAM Usage : {ram.percent}% ({ram.used // (1024**2)}MB / {ram.total // (1024**2)}MB)\n"
        f"Disk Space: {disk.percent}% used ({disk.free // (1024**3)}GB available)"
    )

def read_local_file(filepath):
    """Reads and returns the contents of a local file, expanding ~ to home path."""
    expanded_path = os.path.expanduser(filepath)
    if not os.path.exists(expanded_path):
        return f"Error: File '{filepath}' not found at path '{expanded_path}'."
    try:
        with open(expanded_path, 'r', encoding='utf-8') as f:
            content = f.read(2000)
        return content
    except Exception as e:
        return f"Error reading file: {e}"

def execute_terminal_cmd(command):
    """Safely executes a terminal command on the host OS with extended 60s timeout."""
    try:
        # Timeout extended to 60 seconds
        result = subprocess.run(command, shell=True, capture_output=True, text=True, timeout=60)
        output = result.stdout.strip() if result.stdout else result.stderr.strip()
        return output if output else "Command executed successfully with no output."
    except subprocess.TimeoutExpired:
        return "Execution Error: Command timed out after 60 seconds. It may be running indefinitely or waiting for user input."
    except Exception as e:
        return f"Execution Error: {e}"

def query_ollama(prompt):
    """Sends prompt to the local Ollama instance."""
    payload = {
        "model": MODEL_NAME,
        "prompt": f"{SYSTEM_PROMPT}\n\n[Operator]: {prompt}\n[{DYNA_NAME}]:",
        "stream": False
    }
    try:
        response = requests.post(OLLAMA_URL, json=payload, timeout=300)
        if response.status_code == 200:
            return response.json().get("response", "").strip()
        else:
            return f"API Error (Status {response.status_code}): Ensure model '{MODEL_NAME}' is loaded."
    except requests.exceptions.Timeout:
        return f"Timeout Error: '{MODEL_NAME}' took too long to respond."
    except requests.exceptions.ConnectionError:
        return "Connection Error: Cannot reach Ollama at http://localhost:11434."
    except Exception as e:
        return f"Unexpected Error: {e}"

def main():
    global session_command_history
    print("=" * 60)
    print(f"[{DYNA_NAME} Core] Initializing local cognitive connection...")
    print(f"[{DYNA_NAME} Shell] Connected to model: {MODEL_NAME}")
    print("Commands: 'sysinfo' | 'history' | 'read <file>' | 'run <cmd>' | 'exit'")
    print("=" * 60 + "\n")

    while True:
        try:
            user_input = input("[\033[1;32mOperator\033[0m]: ").strip()
            
            if not user_input:
                continue
                
            if user_input.lower() in ['exit', 'quit']:
                print(f"[{DYNA_NAME} Kernel]: Terminating session. Goodbye, Operator.")
                break
                
            # 1. System Diagnostics
            if user_input.lower() in ['sysinfo', 'status']:
                print(f"\n[\033[1;36m{DYNA_NAME} Diagnostics\033[0m]:")
                print(get_system_metrics() + "\n")
                continue

            # 2. View and Recall Command History
            if user_input.lower() == 'history':
                if not session_command_history:
                    print(f"\n[\033[1;36m{DYNA_NAME} History\033[0m]: No commands suggested yet in this session.\n")
                else:
                    print(f"\n[\033[1;36m{DYNA_NAME} Session Command History\033[0m]:")
                    for idx, cmd in enumerate(session_command_history, 1):
                        print(f"  [{idx}] {cmd}")
                    
                    choice = input("\nSelect a command number to execute (or press Enter to go back): ").strip()
                    if choice.isdigit():
                        idx = int(choice) - 1
                        if 0 <= idx < len(session_command_history):
                            selected_cmd = session_command_history[idx]
                            print(f"\n[\033[1;33mExecuting Recalled Command\033[0m]: {selected_cmd}")
                            output = execute_terminal_cmd(selected_cmd)
                            print(f"[\033[1;36mOutput\033[0m]:\n{output}\n")
                continue

            # 3. Local File Reader Tool
            if user_input.lower().startswith('read '):
                filepath = user_input[5:].strip()
                print(f"[\033[1;36m{DYNA_NAME}\033[0m]: Reading file '{filepath}'...")
                file_data = read_local_file(filepath)
                prompt = f"Analyze the following file content from '{filepath}':\n\n{file_data}"
                response = query_ollama(prompt)
                print(f"\n[\033[1;36m{DYNA_NAME}\033[0m]: {response}\n")
                continue

            # 4. Direct Command Execution Tool
            if user_input.lower().startswith('run '):
                cmd = user_input[4:].strip()
                print(f"[\033[1;33mExecuting Command\033[0m]: {cmd}")
                output = execute_terminal_cmd(cmd)
                print(f"[\033[1;36mOutput\033[0m]:\n{output}\n")
                continue

            # 5. Standard AI Query
            print(f"[\033[1;36m{DYNA_NAME}\033[0m]: Processing...", end="\r")
            response = query_ollama(user_input)
            print(" " * 50, end="\r")
            print(f"[\033[1;36m{DYNA_NAME}\033[0m]: {response}\n")

            # Extract any backticked commands and save them to session history
            extracted_cmds = re.findall(r'`([^`]+)`', response)
            for cmd in extracted_cmds:
                if cmd not in session_command_history:
                    session_command_history.append(cmd)
            
            # Interactive prompt for newly suggested commands
            if extracted_cmds:
                print("[\033[1;33mDYNA Interactive Prompt\033[0m]: Detected executable commands in response.")
                for idx, cmd in enumerate(extracted_cmds, 1):
                    print(f"  [{idx}] {cmd}")
                
                choice = input("Execute one of these commands? Enter number (1-%d) or press Enter to skip: " % len(extracted_cmds)).strip()
                if choice.isdigit():
                    idx = int(choice) - 1
                    if 0 <= idx < len(extracted_cmds):
                        selected_cmd = extracted_cmds[idx]
                        print(f"\n[\033[1;33mExecuting Selected Command\033[0m]: {selected_cmd}")
                        output = execute_terminal_cmd(selected_cmd)
                        print(f"[\033[1;36mOutput\033[0m]:\n{output}\n")

        except KeyboardInterrupt:
            print(f"\n[{DYNA_NAME} Kernel]: Session interrupted by operator.")
            sys.exit(0)

if __name__ == "__main__":
    main()
