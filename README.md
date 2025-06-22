</head>
<body class="bg-gray-100">
    <div class="markdown-body">
        <h1 class="text-center"><strong>Self-Hosted Ad-Blocking DNS with AdGuard Home & Docker</strong></h1>
        <p class="text-center">
          <em>A comprehensive guide to setting up a personal, secure, and powerful network-wide ad-blocking DNS server on a cloud VPS.</em>
        </p>
        <hr>
        <h2><strong>Table of Contents</strong></h2>
        <ol>
            <li><a href="#project-overview"><strong>Project Overview</strong></a></li>
            <li><a href="#prerequisites"><strong>Prerequisites</strong></a></li>
            <li><a href="#phase-1-dns--firewall-setup"><strong>Phase 1: DNS & Firewall Setup</strong></a></li>
            <li><a href="#phase-2-server-preparation"><strong>Phase 2: Server Preparation</strong></a></li>
            <li><a href="#phase-3-adguard-home-deployment"><strong>Phase 3: AdGuard Home Deployment</strong></a></li>
            <li><a href="#phase-4-adguard-home-setup-wizard"><strong>Phase 4: AdGuard Home Setup Wizard</strong></a></li>
            <li><a href="#phase-5-final-configuration"><strong>Phase 5: Final Configuration</strong></a></li>
            <li><a href="#configuring-your-devices"><strong>Configuring Your Devices</strong></a></li>
        </ol>
        <hr>
        <h2 id="project-overview"><strong>Project Overview</strong></h2>
        <p>This project walks you through deploying <strong>AdGuard Home</strong> on a Debian-based cloud VPS using Docker and Docker Compose. This method encapsulates the application, simplifies management, and makes future updates seamless.</p>
        <p><strong>The final directory structure on your server will be:</strong></p>
        <pre><code>/path/to/your/project/
├── docker-compose.yml
├── conf/
│   └── (AdGuard Home configuration files are automatically created here)
└── work/
    └── (AdGuard Home data, logs, and filters are automatically stored here)
</code></pre>
        <p>You only need to manually create the <code>docker-compose.yml</code> file.</p>
        <hr>
        <h2 id="prerequisites"><strong>Prerequisites</strong></h2>
        <ul>
            <li>A <strong>Cloud VPS</strong> running a Debian-based OS (e.g., Debian, Ubuntu).</li>
            <li>A <strong>Static IP Address</strong> assigned to your VPS.</li>
            <li>A <strong>Registered Domain Name</strong> (e.g., <code>yourdomain.com</code>).</li>
            <li>Basic familiarity with the Linux command line.</li>
        </ul>
        <hr>
        <h2 id="phase-1-dns--firewall-setup"><strong>Phase 1: DNS & Firewall Setup</strong></h2>
        <p class="text-center">
            <em>This is the most critical phase. An error here will prevent the setup from working.</em>
        </p>
        <h4><strong>1. Create a DNS 'A' Record</strong></h4>
        <p>You need to point a subdomain (like <code>dns.yourdomain.com</code>) to your VPS's static IP address.</p>
        <ul>
            <li>Log in to your domain registrar's control panel (e.g., GoDaddy, Namecheap, Cloudflare).</li>
            <li>Navigate to the DNS management section for <code>yourdomain.com</code>.</li>
            <li>Create a new <strong><code>A</code> record</strong> with the following details:
                <ul>
                    <li><strong>Host/Name:</strong> <code>dns</code></li>
                    <li><strong>Value/Points to:</strong> <code>&lt;YOUR_VPS_IP&gt;</code></li>
                    <li><strong>TTL (Time To Live):</strong> Set to the lowest possible value (e.g., 1 minute or 600 seconds) for now.</li>
                </ul>
            </li>
            <li><strong>Verification:</strong> After saving, wait a few minutes. Open a terminal and run <code>ping dns.yourdomain.com</code>. It must reply from <code>&lt;YOUR_VPS_IP&gt;</code>.</li>
        </ul>
        <h4><strong>2. Configure Server Firewall Rules</strong></h4>
        <p>You must open the necessary ports on your cloud provider's firewall.</p>
        <ul>
            <li>Log in to your cloud provider's console (e.g., GCP, AWS, DigitalOcean).</li>
            <li>Navigate to the Firewall or Networking security rules section.</li>
            <li>Create a new <strong>ingress (inbound)</strong> rule allowing traffic from <strong>any source (<code>0.0.0.0/0</code>)</strong> to the following ports:
                <ul>
                    <li><strong>TCP:</strong> <code>53, 80, 443, 853</code></li>
                    <li><strong>UDP:</strong> <code>53, 443</code></li>
                </ul>
            </li>
        </ul>
        <hr>
        <h2 id="phase-2-server-preparation"><strong>Phase 2: Server Preparation</strong></h2>
        <h4><strong>1. Connect to Your VPS</strong></h4>
        <pre><code>ssh your_user@&lt;YOUR_VPS_IP&gt;
</code></pre>
        <h4><strong>2. Update Your System</strong></h4>
        <pre><code>sudo apt-get update && sudo apt-get upgrade -y
</code></pre>
        <h4><strong>3. Install Docker and Docker Compose</strong></h4>
        <pre><code># Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker Engine and Compose
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
</code></pre>
        <hr>
        <h2 id="phase-3-adguard-home-deployment"><strong>Phase 3: AdGuard Home Deployment</strong></h2>
        <h4><strong>1. Create Project Directory</strong></h4>
        <p>Create a directory for your project and navigate into it.</p>
        <pre><code>sudo mkdir -p /opt/adguardhome
cd /opt/adguardhome
</code></pre>
        <h4><strong>2. Create <code>docker-compose.yml</code> File</strong></h4>
        <p>Create the file using a text editor like <code>nano</code>.</p>
        <pre><code>sudo nano docker-compose.yml
</code></pre>
        <p>Paste the content from the <code>docker-compose.yml</code> file provided in this repository into the editor. Save and exit (<code>Ctrl+X</code>, <code>Y</code>, <code>Enter</code>).</p>
        <h4><strong>3. Launch AdGuard Home</strong></h4>
        <p>Start the service in the background. Docker will automatically download the AdGuard Home image.</p>
        <pre><code>sudo docker compose up -d
</code></pre>
        <hr>
        <h2 id="phase-4-adguard-home-setup-wizard"><strong>Phase 4: AdGuard Home Setup Wizard</strong></h2>
        <h4><strong>1. Access the Wizard</strong></h4>
        <p>Open a web browser and navigate to <code>http://&lt;YOUR_VPS_IP&gt;:3000</code>.</p>
        <h4><strong>2. Follow On-Screen Instructions</strong></h4>
        <ul>
            <li><strong>Step 1:</strong> Click <strong>Get Started</strong>.</li>
            <li><strong>Step 2 (Admin Web Interface):</strong> Set "Listen Interface" to <strong>All interfaces</strong> and "Port" to <code>80</code>.</li>
            <li><strong>Step 3 (DNS server):</strong> Set "Listen Interface" to <strong>All interfaces</strong> and "Port" to <code>53</code>.</li>
            <li><strong>Step 4 (Authentication):</strong> Create a secure <strong>username</strong> and <strong>password</strong>. <em>Store this password safely!</em></li>
            <li><strong>Step 5 &amp; 6:</strong> Click <strong>Next</strong>, then <strong>Open Dashboard &amp; log in</strong>.</li>
        </ul>
        <p>You will be redirected to <code>http://&lt;YOUR_VPS_IP&gt;</code>. Log in with your new credentials.</p>
        <hr>
        <h2 id="phase-5-final-configuration"><strong>Phase 5: Final Configuration</strong></h2>
        <h4><strong>1. Enable Encryption (HTTPS/DoH/DoT)</strong></h4>
        <p>This is the final step to secure your server.</p>
        <ul>
            <li>In the AdGuard dashboard, navigate to <strong>Settings -> Encryption settings</strong>.</li>
            <li>Check <strong>Enable encryption</strong>.</li>
            <li><strong>Server name:</strong> Enter <code>dns.yourdomain.com</code>.</li>
            <li>Check <strong>Redirect to HTTPS automatically</strong>.</li>
            <li><strong>Certificate source:</strong> Select <strong>Use AdGuard Home to get a certificate from Let's Encrypt</strong>.</li>
            <li>Enter your email address and click <strong>Save configuration</strong>.</li>
        </ul>
        <p>AdGuard will now obtain an SSL certificate. Your browser will reload, and you will be at <code>https://dns.yourdomain.com</code>.</p>
        <h4><strong>2. Secure Your Server (IMPORTANT!)</strong></h4>
        <p>Now that the wizard is complete, you <em>must</em> close the setup port (<code>3000</code>) for security.</p>
        <ul>
            <li>Go back to your SSH terminal.</li>
            <li>Edit the <code>docker-compose.yml</code> file: <code>sudo nano /opt/adguardhome/docker-compose.yml</code></li>
            <li><strong>Delete or comment out (<code>#</code>)</strong> the line <code>- "3000:3000/tcp"</code>.</li>
            <li>Save the file and apply the changes:
                <pre><code>sudo docker compose up -d
</code></pre>
            </li>
        </ul>
        <h4><strong>3. Configure Upstream DNS & Blocklists</strong></h4>
        <ul>
            <li>In the dashboard, go to <strong>Settings -> DNS settings</strong>.</li>
            <li>Under <strong>Upstream DNS servers</strong>, add secure resolvers like:
                <ul>
                    <li><code>https://dns.quad9.net/dns-query</code></li>
                    <li><code>https://dns.cloudflare.com/dns-query</code></li>
                </ul>
            </li>
            <li>Go to <strong>Filters -> DNS blocklists</strong> and add popular lists to enhance ad-blocking.</li>
        </ul>
        <hr>
        <h2 id="configuring-your-devices"><strong>Configuring Your Devices</strong></h2>
        <p>Your private, secure DNS server is ready! Use the following addresses to configure your devices.</p>
        <ul>
            <li><strong>DNS-over-TLS (DoT):</strong> (Recommended for Android)
                <ul>
                    <li><code>dns.yourdomain.com</code></li>
                </ul>
            </li>
            <li><strong>DNS-over-HTTPS (DoH):</strong> (Recommended for browsers, Windows, and Apple devices)
                <ul>
                    <li><code>https://dns.yourdomain.com/dns-query</code></li>
                </ul>
            </li>
            <li><strong>Plain DNS:</strong>
                <ul>
                    <li><code>&lt;YOUR_VPS_IP&gt;</code></li>
                </ul>
            </li>
        </ul>
    </div>
</body>
</html>
