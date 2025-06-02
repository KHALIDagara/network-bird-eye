
Download latest Prometheus (Linux AMD64)
wget https://github.com/prometheus/prometheus/releases/download/v2.45.0/prometheus-2.45.0.linux-amd64.tar.gz

# Extract it
tar xvfz prometheus-2.45.0.linux-amd64.tar.gz

# Move to a clean directory (optional)
cd prometheus-2.45.0.linux-amd64

# Copy your config file to this directory
cp ../prometheus.yml . && ./prometheus

# this script does very simple thing .. downloads the binary of prometheus ..2 - extracts it .. 3-cd into the binary folder and copies the config file 
# prometheus.yml to the binary folder and run the binary with the command ./prometheus 

