# Flow Watermarking Attack on Tor

This repository contains the code and paper for implementing and documenting a flow watermarking attack on Tor, based on the interval centroid technique described by Wang et al.

## Repository Structure

```
flow-watermarking-attack/
├── paper/                      # LaTeX paper and build files
│   ├── main.tex                # Main LaTeX document
│   ├── threatmodel.png         # Threat model diagram
│   └── build.sh                # Build script for paper
├── code/                       # Python implementation of the attack
│   ├── watermark_common.py     # Shared watermarking code
│   ├── watermark_encoder.py    # Watermark encoder for entry node
│   ├── watermark_detector.py   # Watermark detector for exit node
│   └── analyze_results.py      # Analysis script for results
└── README.md                   # This README file
```

## Setup Instructions

### 1. Install Dependencies

Install the required dependencies for both the LaTeX paper and the Python code:

```bash
# For LaTeX
sudo apt-get update
sudo apt-get install -y texlive-latex-base texlive-latex-extra texlive-science \
                        texlive-bibtex-extra biber latexmk

# For Python code
sudo apt-get install -y python3-pip python3-dev libpcap-dev
pip3 install scapy numpy matplotlib
```

### 2. Set Up Experiment Environment

#### Configure Tor on the Raspberry Pi nodes:

1. **Entry Node (169.254.0.8):**
   ```bash
   sudo apt-get install -y tor
   # Edit /etc/tor/torrc to configure as entry/guard node
   sudo systemctl restart tor
   ```

2. **Exit Node (169.254.0.9):**
   ```bash
   sudo apt-get install -y tor
   # Edit /etc/tor/torrc to configure as exit node
   sudo systemctl restart tor
   ```

3. **Client Node (169.254.0.1):**
   ```bash
   sudo apt-get install -y tor
   # Configure torrc to use specific entry and exit nodes
   sudo systemctl restart tor
   ```

4. **Server Node (169.254.0.10):**
   ```bash
   # Set up the Flask server
   sudo apt-get install -y python3-flask
   ```

### 3. Deploy the Attack Code

1. **On Entry Node:**
   Copy the following files to the entry node:
   - `watermark_common.py`
   - `watermark_encoder.py`

   Make the script executable:
   ```bash
   chmod +x watermark_encoder.py
   ```

2. **On Exit Node:**
   Copy the following files to the exit node:
   - `watermark_common.py`
   - `watermark_detector.py`

   Make the script executable:
   ```bash
   chmod +x watermark_detector.py
   ```

3. **On Analysis Computer:**
   Copy all Python files for later analysis.

### 4. Running the Attack

#### 1. Generate the Web Traffic

On the server node (169.254.0.10), create HTML files for testing:

```bash
mkdir -p ~/www
cat > ~/www/small.html << EOF
<!DOCTYPE html>
<html><head><title>Small Test Page</title></head>
<body><h1>Small Test Page</h1><p>This is a small test page</p></body>
</html>
EOF

cat > ~/www/medium.html << EOF
<!DOCTYPE html>
<html><head><title>Medium Test Page</title></head>
<body>
<h1>Medium Test Page</h1>
<p>This is a medium-sized test page with multiple paragraphs.</p>
<p>It contains more content than the small page.</p>
<div>Additional content to make the page larger</div>
</body>
</html>
EOF

cat > ~/www/large.html << EOF
<!DOCTYPE html>
<html><head><title>Large Test Page</title></head>
<body>
<h1>Large Test Page</h1>
<p>This is a large test page with significant content.</p>
<!-- Add more content to make the page larger -->
</body>
</html>
EOF

cd ~/www
sudo python3 -m http.server 80
```

#### 2. Start the Watermark Detector

On the exit node (169.254.0.9):

```bash
cd ~/attack
sudo python3 watermark_detector.py --target 169.254.0.10 --iface eth0 --seed 12345 --window 300
```

#### 3. Start the Watermark Encoder

On the entry node (169.254.0.8):

```bash
cd ~/attack
sudo python3 watermark_encoder.py --target 169.254.0.1 --iface eth0 --seed 12345
```

#### 4. Generate Traffic Through Tor

On the client node (169.254.0.1):

```bash
# Generate traffic through Tor
for i in {1..20}; do
  torsocks curl http://169.254.0.10/small.html > /dev/null
  sleep 2
  torsocks curl http://169.254.0.10/medium.html > /dev/null
  sleep 2
  torsocks curl http://169.254.0.10/large.html > /dev/null
  sleep 2
done
```

#### 5. Analyze the Results

After the experiment, collect the log files from the entry and exit nodes:

```bash
scp pi@169.254.0.8:~/attack/watermark_encoder.log .
scp pi@169.254.0.9:~/attack/watermark_detector.log .

python3 analyze_results.py --encoder-log watermark_encoder.log --detector-logs watermark_detector.log
```

## Troubleshooting

Common issues and solutions:

1. **Permission Issues with Scapy**:
   - The Python scripts must be run as root (using sudo) because they use raw sockets for packet capture.

2. **Tor Configuration Issues**:
   - Check Tor logs at `/var/log/tor/notices.log` if circuits aren't forming properly.

3. **Packet Capture Problems**:
   - Verify the interface name with `ip a` command.
   - Test packet capture with `sudo tcpdump -i <interface> host <target-ip>`.

4. **LaTeX Build Errors**:
   - Make sure all required LaTeX packages are installed.
   - Check for syntax errors in the LaTeX file.

