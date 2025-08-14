# Omnisend v3 Contact Updater

A PHP script that updates existing Omnisend contacts from a CSV file using the **Omnisend v3 API**.  
Specifically designed to transfer the **Interests** subscriber preference from Mailchimp, since Omnisend currently doesn’t allow importing it via their normal CSV importer.

---

## 📋 Use Case

When migrating from **Mailchimp** to **Omnisend**, subscriber **Interests** cannot be imported via Omnisend's standard CSV importer.  
This script uses the Omnisend v3 API to update these properties in bulk, preserving your segmentation data without manual work.

---

## ✨ Features

- Updates **only** existing contacts — never creates new ones  
- Preserves subscription status  
- Supports `customProperties` as arrays (e.g., multiple interests)  
- Logs **updated** contacts to `changed.csv`  
- Logs **skipped** contacts to `skipped.csv`  
- Simple CSV format for easy data migration

---

## 📂 CSV Format

Your CSV should have **two columns**:  

```csv
email,interests
user1@example.com,"Ladies,Gentlemen"
user2@example.com,Ladies
user3@example.com,"Ladies,Gentlemen,Children"
```

Quotes are required when the interests column contains multiple values.
Single values can remain unquoted.

---

## ⚙️ Setup

- Clone this repo or download the script.
- Save your contacts file as contacts.csv in the same folder as the script.
- Get your Omnisend API key from your account (Account → API Keys).

Edit the script and set:
- $apiKey = 'YOUR_OMNISEND_API_KEY';
- Make sure PHP is installed (7.4+) with curl enabled.

---

## ▶️ Usage

Run the script from the terminal:

php update_omisend_contacts_with_iterests.php

Example output:

Skipping header row...
[2] Searching: user1@example.com
 → Found contactID: 123abc456. Updating interests...
   PATCH status: 200
[3] Searching: notfound@example.com
   ℹ Contact not found. Skipping.

Updated contacts will be saved to changed.csv
Skipped contacts will be saved to skipped.csv

---

## 💻 PHP Script
```
<?php
/**
 * Omnisend v3 Contact Updater using customProperties (arrays) to update interests from Mailchimp
 * Only updates existing contacts — never creates new ones.
 * Writes invalid emails and emails not found in omnisend to skipped log
 * Writes changed emails to log
 *
 * CSV format:
 * email,interests
 * email@example.com,"Dam|Herr"
 *
 * Example debug output:
 * [1] Searching: email@example.com
 * → Found contactID: 6871596079b48cd4df54d728. Updating interests...
 *  PATCH status: 200
 */

$apiKey   = 'YOUR_OMNISEND_API_KEY';
$csvFile      = __DIR__ . '/contacts.csv';
$baseUrl      = 'https://api.omnisend.com/v3/contacts';
$skippedFile  = __DIR__ . '/skipped.csv';
$changedFile  = __DIR__ . '/changed.csv';

// Prepare skipped log with header
$skippedHandle = fopen($skippedFile, 'w');
fputcsv($skippedHandle, ['Email', 'Reason']);

// Prepare changed log with header
$changedHandle = fopen($changedFile, 'w');
fputcsv($changedHandle, ['Email', 'Updated Interests']);

/**
 * Search for a contact by email (v3)
 */
function searchContactByEmail($email, $apiKey, $baseUrl) {
    $url = $baseUrl . '?email=' . urlencode($email);
    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'X-API-KEY: ' . $apiKey,
        'Content-Type: application/json'
    ]);
    $result   = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);

    if ($httpCode === 200) {
        $data = json_decode($result, true);
        if (!empty($data['contacts']) && is_array($data['contacts'])) {
            return $data['contacts'][0];
        }
    }
    return null;
}

/**
 * PATCH a contact by ID (v3) using customProperties
 */
function updateContactById($contactId, $apiKey, $baseUrl, $interests) {
    $url = $baseUrl . '/' . urlencode($contactId);

    $payload = [
        'customProperties' => [
            'interests' => $interests // array of strings
        ]
    ];

    $ch = curl_init($url);
    curl_setopt($ch, CURLOPT_CUSTOMREQUEST, 'PATCH');
    curl_setopt($ch, CURLOPT_POSTFIELDS, json_encode($payload));
    curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
    curl_setopt($ch, CURLOPT_HTTPHEADER, [
        'X-API-KEY: ' . $apiKey,
        'Content-Type: application/json'
    ]);
    $result   = curl_exec($ch);
    $httpCode = curl_getinfo($ch, CURLINFO_HTTP_CODE);
    curl_close($ch);
    return [$httpCode, $result];
}

// ---- MAIN ----
if (!file_exists($csvFile)) {
    exit("CSV file not found: $csvFile\n");
}

if (($handle = fopen($csvFile, 'r')) === false) {
    exit("Unable to open CSV file.\n");
}

$rowNum = 0;
while (($data = fgetcsv($handle)) !== false) {
    $rowNum++;
    if ($rowNum == 1) {
        echo "Skipping header row...\n";
        continue;
    }

    $email = trim($data[0] ?? '');
    // Adjust delimiter based on your CSV: comma or pipe
    $interests = array_filter(array_map('trim', explode(',', $data[1] ?? '')));

    if (!filter_var($email, FILTER_VALIDATE_EMAIL)) {
        echo "[{$rowNum}] Invalid email format, skipping.\n";
        fputcsv($skippedHandle, [$email, 'Invalid email format']);
        continue;
    }

    echo "[{$rowNum}] Searching: {$email}\n";
    $contact = searchContactByEmail($email, $apiKey, $baseUrl);

    if ($contact) {
        $contactId = $contact['contactID'];
        echo " → Found contactID: {$contactId}. Updating interests...\n";
        [$patchCode, $patchBody] = updateContactById($contactId, $apiKey, $baseUrl, $interests);
        echo "   PATCH status: {$patchCode}\n";

        if ($patchCode === 200) {
            // log changed contact
            fputcsv($changedHandle, [$email, implode(',', $interests)]);
        } else {
            echo "   ⚠ Failed to update, skipping.\n";
            fputcsv($skippedHandle, [$email, "PATCH failed ({$patchCode})"]);
        }
    } else {
        echo "   ℹ Contact not found. Skipping.\n";
        fputcsv($skippedHandle, [$email, 'Not found in Omnisend']);
    }
}

fclose($handle);
fclose($skippedHandle);
fclose($changedHandle);

echo "\nSkipped contacts logged in: {$skippedFile}\n";
echo "Updated contacts logged in: {$changedFile}\n";
```
