<p align="center">
    <img src="/images/network-diagram.png" width="1152" alt="Network Diagram">
</p>

## 🧠 Technologies & Skills Used
- **Microsoft Azure Resources:** Network Security Groups, Virtual Machines, Virtual Networks
- **Security Monitoring Tools:** Microsoft Sentinel SIEM, Log Analytics Workspace
- **Log Analysis:** KQL (Kusto Query Language), Security Event Log Analysis
- **Security Concepts:** Honeypot Deployment, Security Baseline Violation (deliberate), Attack Surface Management

<br>

# 📑 Table of Contents
### [🛠️ Creating the Resource Group, VNET, & Virtual Machine](#creating-azure-resources)
### [🍯 Intentionally Making the Virtual Machine Vulnerable](#make-vm-vulnerable)
### [🪵 Exploring Logs](#exploring-logs)
### [📩 Forwarding Logs to Azure Configuration](#forwarding-logs)
### [🔎 Analyzing Log Data with KQL](#kql-analysis)
### [📍 Tracking GeoIP](#tracking-geoip)
### [🌍 World Map of Attacker Location Configuration](#map-setup)
### [🗺️ Visualization of Attack Location Data (Attack Map)](#attack-map)

<br>

<a name="creating-azure-resources"></a>
# 🛠️ Creating Microsoft Azure Resources (RG, VM, VNET)

## 📂 Creating the Resource Group

<img src="/images/create-rg.png" alt="Create RG">

## 🌐 Creating the VNET
**Basics Tab**:
1. Virtual network name: Enter `VNET-SOC-Lab`
2. Resource group: `RG-SOC-Lab`
3. Region: `US East 2`

<img src="/images/create-vnet.png" alt="Create VNET">

## 🖥️ Creating Virtual Machine (Honeypot)
**Basics Tab**:
1. Resource group: `RG-SOC-Lab`
2. Virtual machine name: Enter `CORP-NET-EAST` 
> [!NOTE]
> Avoid using 'honeypot' in the name, as attackers might identify the VM’s purpose and avoid targeting it

3. Region: `US East 2`
4. Image: `Windows 10 Pro, version 22H2 - x64 Gen2`
5. Size: `Standard_D2s_v3 - 2 vcpus, 8 GiB memory`
6. Administrator account: Enter secure credentials
7. Tick ✅`I confirm I have an eligible Windows 10/11 license with multi-tenant hosting rights`

**Disks Tab**:
OS disk type: `Standard HDD (locally-redundant storage)`

**Networking Tab**:
Virtual network: `VNET-SOC-Lab`
Tick ✅`Delete public IP and NIC when VM is deleted`

**Monitoring Tab**:
Boot diagnostics: Tick ✅`Disable`

**Review + Create Tab**:
Click the blue `Create` button

<img src="/images/create-vm.png" alt="Create VM">

<a name="make-vm-vulnerable"></a>
# 🍯 Intentionally Making the Virtual Machine Vulnerable

After the RG, VM, and VNET are created, the populated resource group should look like this—

<img src="/images/populated-rg.png" alt="Populated RG">


## 🚦Delete Current RDP Inbound Rule & Create a Vulnerable One
From here, open up `CORP-NET-EAST-nsg`

Delete the RDP inbound security rule:

<img src="/images/delete-rdp-rule.png" alt="Delete RDP Rule">

On the side panel on the left, navigate to `Settings` > `Inbound security rules` > `+ Add` with the following settings—

Destination port ranges: `*`
Name: `DANGER_AllowAnyCustomAnyInbound`

<img src="/images/create-rdp-rule.png" alt="Create RDP Rule">

## 🧱 Sabotage Windows Defender Firewall Settings

1. Remotely access the `CORP-NET-EAST` virtual machine using the `Remote Desktop Connection` Windows application

2. Open `wf.msc` by searching for it in the search bar. When the Windows Defender Firewall with Advanced Security window appears, click `Windows Defender Firewall Properties`

3. On the **Domain Profile** tab, press the 'O' key and the desired settings will change. Repeat this process for the **Private Profile** and **Public Profile** tabs. Click `Apply` and `OK`

4. Disconnect from the `CORP-NET-EAST` virtual machine

<img src="/images/win-defender.png" alt="Windows Defender">

## 🏓 Ping the Virtual Machine from a Third-party Machine

<img src="/images/ping-vm.png" alt="Ping VM from Third-party Machine">

<a name="exploring-logs"></a>
# 🪵 Exploring Logs

## 🔑 Deliberately Fail RDP Log-in & Observe the Corresponding Logs

1. Attempt to remotely access the `CORP-NET-EAST` virtual machine by using incorrect credentials 4 times

<img src="/images/fail-rdp-login.png" alt="Deliberately Fail RDP Log-in">

2. Remotely access the `CORP-NET-EAST` virtual machine with the legitimate credentials and open `Event Viewer`, and navigate to `Windows Logs` > `Security` > `Find…`

3. In the `Find` pop-up window, search for `4625` which is the event ID for failed log-in attempts

4. Observe the four failed log-in attempts from step 1 by double clicking them (below, it shows that `DREWS_PC` attempted a connection with the IP address of the VPN I'm currently connected to)

<img src="/images/observe-failed-logins.png" alt="Observe Failed Log-ins">

<a name="forwarding-logs"></a>
# 📩 Forwarding Logs to Azure Configuration

1. In `portal.azure.com`, search for `Log Analytics workspaces`, click `+ Create log analytics workspace` button, set the Resource group to `RG-SOC-Lab`, set the Name to `LAW-SOC-Lab-000`, click `Review + Create`, and click `Create` after the validation completes

<img src="/images/create-law.png" alt="Create Log Analytics Workspace">

2. In `portal.azure.com`, search for `Microsoft Sentinel`, click `+ Create Microsoft Sentinel`, add Microsoft Sentinel to the `LAW-SOC-Lab-000` workspace

<img src="/images/link-law-to-sentinel.png" alt="Link LAW to Microsoft Sentinel">

3. In `Microsoft Sentinel`, within the `LAW-SOC-Lab-000` workspace, navigate to `Content management` > `Content hub`, and in the search for `Windows Security Events` in the `Search…` box

4. Tick ✅ `Windows Security Events` and then click the blue `Install` button in the panel on the right

<img src="/images/install-wse.png" alt="Install Windows Security Events">

5. When the install completes, click the blue `Manage` button that's where the `Install` button used to be

6. Tick ✅ `Windows Security Events via AMA` and then click the blue `Open connector page` button in the panel on the right

<img src="/images/open-connector-page.png" alt="Open WSE via AMA Connector Page">

7. Click `+Create data collection rule` and use the following settings

**Basics Tab**:
Set the Rule name to `DCR-Windows` and use the `RG-SOC-Lab` Resource group,

**Resources Tab**:
Expand `Azure subscription 1` > `RG-SOC-Lab` > and tick ✅ `CORP-NET-EAST`

**Review + Create**:
Click the blue `Create` button

8. In the `CORP-NET-EAST` virtual machine, navigate to `Settings` > `Extensions + applications` to observe that `AzureMonitorWindowsAgent` is now in the `Transitioning` phase (eventually it should turn into `Provisioning succeeded`)

<img src="/images/amwa-transitioning.png" alt="AzureMonitorWindowsAgent is Transitioning">

9. In `portal.azure.com`, search for `Log Analytics workspaces`, select `LAW-SOC-Lab-000`, and click on the `Logs` tab, type `SecurityEvent` into the textbox, and click `▶ Run`. At first, there shouldn't be many records. However, after I leave the virtual machine on for around 24 hours, I'll come back and analyze that data with KQL and see how many attacks took place.

<a name="kql-analysis"></a>
# 🔎 Analyzing Log Data with KQL

1. After the 24 hour mark, running this command will let me see how many attacks took place in total:

```kql
    SecurityEvent
    | where EventID == 4625
    | count
```

<img src="/images/kql-attacks-count.png" alt="KQL Attacks Count">

2. Then, these attacks can be queried to display the attacks with only the relevant information such as time generated, account username, the machine that generated the log, event ID, activity (what the event ID means), and IP address:

```kql
    SecurityEvent
    | where EventID == 4625
    | project TimeGenerated, Account, Computer, EventID, Activity, IpAddress
```

<img src="/images/kql-attack-info.png" alt="KQL Attack Info">

<a name="tracking-geoip"></a>
# 📍 Tracking GeoIP

1. In `portal.azure.com`, search for `Microsoft Sentinel`, click `LAW-SOC-Lab-000`, navigate to `Configuration` > `Watchlist`, and click `+ New`

**Basics Tab**:
For the Name and Alias fields, enter `geoip`

**Source Tab**:
Under Upload file, upload `geoip-summarized.csv` and set the SearchKey to `network`

**Review + Create Tab**:
Click the blue `Create` button

<img src="/images/upload-geoip-csv.png" alt="Upload GeoIP .csv">

2. The file will begin to upload (when **Status (Preview)** changes from **♻️Uploading** to **✅Succeeded**, it's time to proceed)

<img src="/images/csv-is-uploading.png" alt=".csv is Uploading">

3. Running the command below will verify that the newly created Watchlist (in the Log Analytics workspace) works properly

```kql
    SecurityEvent

    _GetWatchlist("geoip")
```

<img src="/images/test-geoip-watchlist.png" alt="Test GeoIP Watchlist">

4. Use an attacker’s IP address (from the logs) to query the GeoIP Watchlist and map the attack location:

```kql
    let GeoIPDB_FULL = _GetWatchlist("geoip");
    let WindowsEvents = SecurityEvent
        | where IpAddress == <insert attacker IP address here>
        | where EventID == 4625
        | order by TimeGenerated desc
        | evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network);
    WindowsEvents
    | project TimeGenerated, Computer, AttackerIP = IpAddress, cityname, countryname, latitude, longitude
```

<img src="/images/query-geo-date-of-attacker.png" alt="Query Geo Data of Attacker">

<a name="map-setup"></a>
## 🌍 World Map of Attacker Location Configuration

1. In `portal.azure.com`, search for `Microsoft Sentinel`, click `LAW-SOC-Lab-000`, navigate to `Threat management` > `Workbooks`, click `+ Add Workbook`, and then click `Edit`

2. Remove the default entries that Azure populates the new Workbook with

3. Click `+ Add`, click `Add query`, and then move to the `Advanced Editor` tab

4. Replace the current contents in the `Advanced Editor` textbox with the following .json snippet

```json
{
	"type": 3,
	"content": {
	"version": "KqlItem/1.0",
	"query": "let GeoIPDB_FULL = _GetWatchlist(\"geoip\");\nlet WindowsEvents = SecurityEvent;\nWindowsEvents | where EventID == 4625\n| order by TimeGenerated desc\n| evaluate ipv4_lookup(GeoIPDB_FULL, IpAddress, network)\n| summarize FailureCount = count() by IpAddress, latitude, longitude, cityname, countryname\n| project FailureCount, AttackerIp = IpAddress, latitude, longitude, city = cityname, country = countryname,\nfriendly_location = strcat(cityname, \" (\", countryname, \")\");",
	"size": 3,
	"timeContext": {
		"durationMs": 2592000000
	},
	"queryType": 0,
	"resourceType": "microsoft.operationalinsights/workspaces",
	"visualization": "map",
	"mapSettings": {
		"locInfo": "LatLong",
		"locInfoColumn": "countryname",
		"latitude": "latitude",
		"longitude": "longitude",
		"sizeSettings": "FailureCount",
		"sizeAggregation": "Sum",
		"opacity": 0.8,
		"labelSettings": "friendly_location",
		"legendMetric": "FailureCount",
		"legendAggregation": "Sum",
		"itemColorSettings": {
		"nodeColorField": "FailureCount",
		"colorAggregation": "Sum",
		"type": "heatmap",
		"heatmapPalette": "greenRed"
		}
	}
	},
	"name": "query - 0"
}
```

5. Click `Done Editing`

<img src="/images/done-editing.png" alt="Done Editing">

6. Save the Workbook with the save icon in the toolbar with the Title being `Windows VM Attack Map` and the Resource group being `RG-SOC-Lab` (after clicking the blue `Save As` button, also re-click the save icon in the toolbar)
> [!NOTE]
> The map visualization is highly customizable. For example, the color scheme can be adjusted from Green-Orange-Red to Yellow-Orange-Red

<a name="attack-map"></a>
# 🗺️ Visualization of Attack Location Data (Attack Map)

<img src="/images/attack-map.png" alt="Attack Map">
