# Grafana Dashboard Provisioning

This directory contains JSON exports of Grafana dashboards that will be automatically provisioned when Grafana starts.

If you want to make dashboards appear on new installs, you'll need to place them here.
Dashboards that are provisioned in this folder, also need to be updated here, not in the grafana GUI.

## How to Export Dashboards from Grafana

1. **Open your dashboard in Grafana**
   - Navigate to the dashboard you want to export
   - Click the dashboard settings icon (gear icon) in the top right

2. **Export the dashboard**
   - In the dashboard settings, click **"JSON Model"** in the left sidebar
   - Copy the entire JSON content, OR
   - Click **"Save dashboard"** → **"Export"** → **"Save to file"** to download as JSON

3. **Save the dashboard JSON file**
   - Save the JSON file in this directory (`grafana/provisioning/dashboards/`)
   - Use a descriptive filename, e.g., `node-exporter-full.json`, `docker-monitoring.json`, etc.
   - The file must have a `.json` extension

4. **Restart Grafana**
   - After adding dashboard JSON files, restart the Grafana container:
     ```bash
     docker compose restart grafana
     ```
   - The dashboards will be automatically imported and available in Grafana

## Notes

- Dashboards are automatically provisioned on Grafana startup
- Changes to dashboard JSON files require a Grafana restart to take effect
- You can organize dashboards in subfolders if needed (they will appear as folders in Grafana)
- The `allowUiUpdates: true` setting allows you to edit dashboards in the UI, but changes won't persist if you restart without updating the JSON files
