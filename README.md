# NetApp Daily Health Check for AAP 2.6

This solution performs daily health checks on NetApp ONTAP clusters and StorageGRID grids, generating an HTML report and maintaining CSV historical data.

## What This Solution Does

- Checks health status of 3 ONTAP clusters (version 9.15)
- Checks health status of 1 StorageGRID grid (version 11.9, 15 nodes)
- Generates a consolidated HTML report (suitable for email)
- Appends capacity data to CSV files for historical trending
- Automatically removes CSV entries older than 90 days

## Metrics Collected

### ONTAP
- Cluster and node health status
- Aggregate capacity utilization
- Volume capacity utilization (top 20 by usage)
- SnapMirror relationship status and lag times

### StorageGRID
- Grid health status
- Node health status (all 15 nodes)
- Storage capacity utilization

## File Structure

```
netapp/
├── inventory.yml              # Define your clusters and grids
├── group_vars/
│   └── all.yml                # All configurable settings
├── netapp_health_check.yml    # Main playbook (everything in one file)
└── README.md                  # This file
```

## Prerequisites

Before setting up in AAP 2.6, verify:

1. **Collections installed in execution environment:**
   - `netapp.ontap`
   - `netapp.storagegrid`

2. **Network connectivity:**
   - AAP can reach ONTAP cluster management IPs on port 443
   - AAP can reach StorageGRID admin node on port 443

3. **Credentials ready:**
   - LDAP/AD username and password for ONTAP
   - LDAP/AD username and password for StorageGRID

## AAP 2.6 Setup Instructions

### Step 1: Create Project

1. Navigate to **Resources > Projects**
2. Click **Add**
3. Fill in:
   - **Name:** `NetApp Health Check`
   - **Organization:** Select your organization
   - **Execution Environment:** Select your EE with NetApp collections
   - **Source Control Type:** Git (or Manual if using local files)
   - **Source Control URL:** Your repository URL
4. Click **Save**

### Step 2: Create Credentials

#### ONTAP Credential

1. Navigate to **Resources > Credentials**
2. Click **Add**
3. Fill in:
   - **Name:** `ONTAP Clusters`
   - **Organization:** Select your organization
   - **Credential Type:** Machine (or create custom type)
   - **Username:** Your LDAP/AD username
   - **Password:** Your LDAP/AD password
4. Click **Save**

#### StorageGRID Credential

1. Navigate to **Resources > Credentials**
2. Click **Add**
3. Fill in:
   - **Name:** `StorageGRID`
   - **Organization:** Select your organization
   - **Credential Type:** Machine (or create custom type)
   - **Username:** Your LDAP/AD username
   - **Password:** Your LDAP/AD password
4. Click **Save**

**Note:** The playbook authenticates to StorageGRID using the `/api/v3/authorize` endpoint to obtain an API token, then uses that token for all subsequent API calls.

### Step 3: Create Inventory

1. Navigate to **Resources > Inventories**
2. Click **Add > Add inventory**
3. Fill in:
   - **Name:** `NetApp Systems`
   - **Organization:** Select your organization
4. Click **Save**
5. Click **Sources** tab
6. Click **Add**
7. Fill in:
   - **Name:** `NetApp Inventory Source`
   - **Source:** Sourced from a Project
   - **Project:** `NetApp Health Check`
   - **Inventory file:** `inventory.yml`
8. Click **Save**
9. Click **Sync** to import the inventory

### Step 4: Create Job Template

1. Navigate to **Resources > Templates**
2. Click **Add > Add job template**
3. Fill in:
   - **Name:** `NetApp Daily Health Check`
   - **Job Type:** Run
   - **Inventory:** `NetApp Systems`
   - **Project:** `NetApp Health Check`
   - **Execution Environment:** Your EE with NetApp collections
   - **Playbook:** `netapp_health_check.yml`
   - **Credentials:**
     - Add `ONTAP Clusters`
     - Add `StorageGRID`
4. Under **Variables** (YAML format):
   ```yaml
   # ONTAP credentials
   ontap_username: "{{ lookup('env', 'ONTAP_USERNAME') | default(ansible_user) }}"
   ontap_password: "{{ lookup('env', 'ONTAP_PASSWORD') | default(ansible_password) }}"
   # StorageGRID credentials (used for API authentication to get token)
   storagegrid_username: "{{ lookup('env', 'STORAGEGRID_USERNAME') | default(ansible_user) }}"
   storagegrid_password: "{{ lookup('env', 'STORAGEGRID_PASSWORD') | default(ansible_password) }}"
   ```
5. Click **Save**

**Note:** The credential variables depend on your credential type configuration. You may need to adjust the variable names based on how your credentials inject variables. The playbook will authenticate to StorageGRID using these credentials to obtain an API token.

### Step 5: Create Schedule

1. From the job template page, click **Schedules** tab
2. Click **Add**
3. Fill in:
   - **Name:** `Daily 7am`
   - **Start date/time:** Select today, 07:00
   - **Repeat frequency:** Day
   - **Every:** 1 day
4. Click **Save**

## Configuration Options

Edit `group_vars/all.yml` to customize:

| Setting | Default | Description |
|---------|---------|-------------|
| `capacity_warning_threshold` | 80 | Yellow warning at this % used |
| `capacity_critical_threshold` | 90 | Red critical at this % used |
| `snapmirror_lag_warning_hours` | 4 | SnapMirror lag warning (hours) |
| `snapmirror_lag_critical_hours` | 24 | SnapMirror lag critical (hours) |
| `report_output_dir` | `/var/lib/awx/netapp_reports` | Where to save reports |
| `csv_retention_days` | 90 | Days to keep CSV history |

## Output Files

After each run, the following files are created/updated in the report directory:

| File | Description |
|------|-------------|
| `netapp_health_report.html` | Daily HTML report (overwritten each run) |
| `ontap_capacity_history.csv` | ONTAP aggregate capacity history |
| `storagegrid_capacity_history.csv` | StorageGRID capacity history |
| `snapmirror_history.csv` | SnapMirror relationship history |

## CSV File Formats

### ontap_capacity_history.csv
```
date,cluster_name,aggregate_name,total_gb,used_gb,available_gb,used_percent
2026-01-16,Production-Cluster,aggr1,10240.0,7168.0,3072.0,70.0
```

### storagegrid_capacity_history.csv
```
date,grid_name,total_tb,used_tb,used_percent
2026-01-16,Object-Storage-Grid,500.0,125.0,25.0
```

### snapmirror_history.csv
```
date,cluster_name,source_path,destination_path,state,healthy,lag_seconds
2026-01-16,Production-Cluster,svm1:vol1,svm2:vol1_dr,snapmirrored,True,PT30M
```

## Emailing the Report

To email the HTML report after generation, you can:

1. Add a notification template in AAP that includes the report as an attachment
2. Or add an additional task to the playbook using `community.general.mail` module

Example mail task (add to end of playbook if needed):
```yaml
- name: Email health report
  community.general.mail:
    host: smtp.example.com
    port: 25
    to: storage-team@example.com
    subject: "NetApp Daily Health Report - {{ report_date }}"
    body: "See attached HTML report for details."
    attach: "{{ report_output_dir }}/{{ html_report_filename }}"
```

## Verification Checklist

Before first run, verify:

- [ ] Inventory updated with correct hostnames/IPs
- [ ] ONTAP credentials work (test with `ansible-playbook --check`)
- [ ] StorageGRID credentials work
- [ ] Report output directory is writable
- [ ] Collections are in the execution environment
- [ ] Schedule is enabled

## Troubleshooting

### Connection Issues
- Verify network connectivity to management IPs
- Check firewall rules allow HTTPS (443)
- Verify credentials are correct

### Module Errors
- Ensure `netapp.ontap` and `netapp.storagegrid` collections are installed
- Check execution environment includes required Python packages

### Report Not Generated
- Check the `report_output_dir` exists and is writable
- Review job output for errors

### CSV Files Growing Too Large
- Adjust `csv_retention_days` in group_vars
- Manually clean old entries if needed

## Extending This Solution

This playbook is designed as a starting point. Possible enhancements:

- Add email notifications
- Add more ONTAP metrics (SVM health, network ports, etc.)
- Add StorageGRID tenant bucket usage
- Create Grafana dashboards from CSV data
- Add alerting for critical conditions
