# Power BI Guides

# Table of Contents

1. [Introduction](#introduction)  
2. [Getting Started](#getting-started)  
   - [Publishing Reports](#publishing-reports)  
   - [Power BI Service Overview](#power-bi-service-overview)  
3. [Automating Email Reports](#automating-email-reports)  
   - [Option 1: Power BI Subscriptions](#option-1)  
   - [Option 2: Export to Excel/PDF with Power Automate](#option-2)  
   - [Which Method Should You Use?](#which-method)  
   - [Workflow Diagram](#workflow-diagram)  
4. [Data Refresh Scheduling](#data-refresh-scheduling)  
5. [Best Practices](#best-practices)  
6. [Troubleshooting](#troubleshooting)  
7. [Resources](#resources)

---

<a id="introduction"></a>
# Introduction

This guide provides step-by-step instructions for automating tasks in Power BI, setting up email reports, scheduling refreshes, and following best practices.

---

<a id="getting-started"></a>
# Getting Started

<a id="publishing-reports"></a>
## Publishing Reports

Instructions for publishing Power BI reports to the Service will go here.

<a id="power-bi-service-overview"></a>
## Power BI Service Overview

Overview of the Power BI Service environment, navigation, and key features.

---

<a id="automating-email-reports"></a>
# Automating Email Reports

You can set up Power BI to send out daily email reports based on a table or visual using **Subscriptions** or by automating with **Power Automate**.

<a id="option-1"></a>
## Option 1: Power BI Subscriptions (Snapshot in Email)

Use this method if you only need a **snapshot image** of a table or report page delivered by email.

1. **Publish your report** to the Power BI Service (`https://app.powerbi.com`).
2. Open the report in the Service.
3. Go to the page or table visual you want emailed.
4. In the top menu, click **Subscribe** → **Add new subscription**.
5. Configure:
   - **Recipients** – yourself and/or other licensed users  
   - **Subject line & message** – customize the email text  
   - **Schedule** – set to **Daily** and choose a delivery time  
   - **Content** – entire report page or specific visual
6. Save – Power BI will now email a snapshot each day.

> ⚠️ **Note:** Recipients must have access to the workspace or app unless you are using Power BI Premium Per User (PPU) or Premium capacity with external user support.

---

<a id="option-2"></a>
## Option 2: Export to Excel/PDF + Email (Automation with Power Automate)

Use this method if you need the **actual table data** delivered as a file attachment.

1. In the Power BI Service, go to your workspace → Dataset → **Schedule Refresh** to ensure the data is updated daily.
2. Open the report and click **More options (…) → Automate → Power Automate**.
3. Choose a template such as:
   - **Export to Excel and email**
   - **Export report to PDF and email**
4. Configure the flow:
   - **Trigger:** Recurrence (e.g., every day at 8:00 AM)  
   - **Action:** Export the report/page/table  
   - **Action:** Send email with the file attached
5. Save the flow – Power Automate will now deliver a fresh Excel or PDF copy each day.

---

<a id="which-method"></a>
## Which Method Should You Use?

- **Snapshot in Email Body (image only)** → Use **Subscriptions**  
- **Downloadable Table Data (Excel/PDF)** → Use **Power Automate**

---

<a id="workflow-diagram"></a>
## Workflow Diagram

*(Insert diagram image here)*

---

<a id="data-refresh-scheduling"></a>
# Data Refresh Scheduling

Details about scheduling dataset refreshes in the Power BI Service.

---

<a id="best-practices"></a>
# Best Practices

Recommended guidelines for designing, publishing, and sharing reports.

---

<a id="troubleshooting"></a>
# Troubleshooting

Common issues and fixes for Power BI email automation and refresh scheduling.

---

<a id="resources"></a>
# Resources

Helpful links, Microsoft docs, and community resources.
