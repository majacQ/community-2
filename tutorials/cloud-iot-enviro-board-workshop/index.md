---
title: Sensor data collection and analytics with IoT Core
description: Deploy an end-to-end monitoring and analysis setup with a Coral Environmental Sensor Board, IoT Core, and BigQuery. Visualize and analyze data with Google Sheets.
author: kingman
tags: Internet of Things, Raspberry Pi
date_published: 2019-09-25
---

Charlie Wang | Solutions Architect | Google

<p style="background-color:#CAFACA;"><i>Contributed by Google employees.</i></p>

This tutorial shows you how to monitor an environment and conduct an ad-hoc analysis on the collected data. 

In this tutorial, you do the following:

*   Set up a [Coral Environmental Sensor Board](https://coral.withgoogle.com/products/environmental) with a
    [Raspberry Pi Zero W](https://www.raspberrypi.org/products/raspberry-pi-zero-w/) with a 40-pin header.
*   Set up Cloud Functions for sensor data processing.
*   Set up BigQuery for data storage.
*   Set up Google Sheets to integrate with BigQuery and IoT Core.
*   Collect sensor data from the Coral Environmental Sensor Board and process the data on Google Cloud.

The architecture is shown in the following diagram:

![high-level overview](https://storage.googleapis.com/gcp-community/tutorials/cloud-iot-enviro-board-workshop/architecture.png)

In this tutorial, the sensor data is generated by the Coral Environmental Sensor Board and streamed to
[IoT Core](https://cloud.google.com/iot-core/) over the [MQTT](http://mqtt.org/) protocol. The sensor data is 
automatically published on [Pub/Sub](https://cloud.google.com/pubsub/) by IoT Core. Data published on
Pub/Sub automatically triggers a [Cloud Function](https://cloud.google.com/functions/) that processes the data and 
stores it in [BigQuery](https://cloud.google.com/bigquery/). The data is loaded into
[Google Sheets](https://docs.google.com/spreadsheets) for analysis using the
[Sheets data connector for BigQuery](https://cloud.google.com/blog/products/g-suite/connecting-bigquery-and-google-sheets-to-help-with-hefty-data-analysis).
To control the Coral Environmental Sensor Board, command messages are sent from the spreadsheet using the [IoT Core API](https://cloud.google.com/iot/docs/how-tos/commands).

## Prerequisites

*   A [Coral Environmental Sensor Board](https://coral.withgoogle.com/products/environmental), which can be purchased from 
    [Mouser](https://www.mouser.com/ProductDetail/Coral/G650-04023-01?qs=sGAEpiMZZMve4%2FbfQkoj%252BLRlYTc0g%252BeK1GNpvUlp1KI%3D).
*   A [Raspberry Pi Zero W](https://www.raspberrypi.org/products/raspberry-pi-zero-w/) with a 40-pin header and the
    [Raspbian](https://www.raspberrypi.org/downloads/) operating system installed. The Raspberry Pi must be connected to the
    internet. (This tutorial uses the Raspberry Pi Zero W board, but you can use this tutorial with any Raspberry Pi model 
    with a 40-pin header.)
*   A [GCP account](https://console.cloud.google.com/freetrial).
*   A user account linked to [G Suite Business, Enterprise, or Education](https://gsuite.google.com/) for accessing the
    [Sheets data connector for BigQuery](https://cloud.google.com/blog/products/g-suite/connecting-bigquery-and-google-sheets-to-help-with-hefty-data-analysis).

## Costs

This tutorial uses billable components of Google Cloud, including the following:
*   [IoT Core](https://cloud.google.com/iot/pricing)
*   [Pub/Sub](https://cloud.google.com/pubsub/pricing)
*   [Cloud Functions](https://cloud.google.com/functions/pricing)
*   [BigQuery](https://cloud.google.com/bigquery/pricing)

This tutorial should not generate any usage that would not be covered by the [free tier](https://cloud.google.com/free/). 
You can use the [Pricing Calculator](https://cloud.google.com/products/calculator/#id=ceceb3a3-56ec-411d-9a10-91317bc7255b)
to generate a cost estimate based on your projected production usage.

## Before you begin

1.  [Select or create a GCP project](https://console.cloud.google.com/projectselector2/home/dashboard).
1.  Make sure that [billing is enabled](https://cloud.google.com/billing/docs/how-to/modify-project) for your GCP project.
1.  [Enable the IoT, Cloud Functions, Pub/Sub, and BigQuery APIs](https://console.cloud.google.com/flows/enableapi?apiid=cloudiot.googleapis.com,pubsub.googleapis.com,cloudfunctions.googleapis.com,bigquery-json.googleapis.com).

When you finish this tutorial, you can avoid continued billing by deleting the resources you created. For more information,
see [Cleaning up](#Cleaning-up).

## Set up the devices

In this section, you set up the Coral Environmental Sensor Board and Raspberry Pi.

### Attach the devices

Attach the Coral Environmental Sensor Board to the 40-pin header of the Raspberry Pi and power on the Raspberry Pi by 
plugging the power cable into it.

![board setup](https://storage.googleapis.com/gcp-community/tutorials/cloud-iot-enviro-board-workshop/board-setup.jpg)

### Install the library and driver

Install the Coral Environmental Sensor Board library and driver on the Raspberry Pi:

1.  Follow the
    [instructions for installing the Python library](https://coral.withgoogle.com/docs/enviro-board/get-started/#install-the-python-library)
    on the Coral Environmental Sensor Board official page.
1.  Reboot the Raspberry Pi board.

### Download the tutorial source code to the board

In the Raspberry Pi shell, run the following command to download the necessary source code to the Raspberry Pi:

    mkdir -p "$HOME"/enviro-board
    cd "$HOME"/enviro-board
    wget https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/cloud-iot-enviro-board-workshop/enviro-device/cloud_config_template.ini
    wget https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/cloud-iot-enviro-board-workshop/enviro-device/core.py
    wget https://raw.githubusercontent.com/GoogleCloudPlatform/community/master/tutorials/cloud-iot-enviro-board-workshop/enviro-device/enviro_demo.py

### Get the public key of the secure element of the sensor board

1.  Run the following command in the Raspberry Pi shell to get the
    [cryptoprocessor](https://coral.withgoogle.com/docs/enviro-board/datasheet/#secure-cryptoprocessor) public key of
    the Coral Environmental Sensor Board: 

        cd /usr/lib/python3/dist-packages/coral/cloudiot

        python3 ecc608_pubkey.py

1.  Copy the public key and save it so that you can use it later, when creating the device identity in IoT Core.

## Get the tutorial source code in Cloud Shell

1.  In the GCP Console, [open Cloud Shell](http://console.cloud.google.com/?cloudshell=true).
1.  Clone the source code repository:

        cd $HOME
        git clone https://github.com/GoogleCloudPlatform/community.git

## Create the public key file for the sensor board

The cryptoprocessor exposes an elliptic curve public key, which you got in a previous section. In the current section,
you save the public key in Cloud Shell in [PEM](https://en.wikipedia.org/wiki/Privacy-Enhanced_Mail) format and wrap the key
with `-----BEGIN PUBLIC KEY-----` and `-----END PUBLIC KEY-----`.

1.  Create the key file in Cloud Shell and name it `device_pub_key.pem`:

        touch $HOME/community/tutorials/cloud-iot-enviro-board-workshop/device_pub_key.pem

1.  Add the start wrapper to the key file:

        echo '-----BEGIN PUBLIC KEY-----' >> $HOME/community/tutorials/cloud-iot-enviro-board-workshop/device_pub_key.pem

1.  Add the public key to the key file:

        echo 'replace with public key string' >> $HOME/community/tutorials/cloud-iot-enviro-board-workshop/device_pub_key.pem

1.  Add the end wrapper to the key file:

        echo '-----END PUBLIC KEY-----' >> $HOME/community/tutorials/cloud-iot-enviro-board-workshop/device_pub_key.pem

## Set up the device identity on GCP

For a device to communicate with IoT Core, the device identity needs to be created in IoT Core.

1.  To make it easier to run commands when you create cloud resources, set environment variables in Cloud Shell to hold the
    names and properties of the resources:

        export EVENT_TOPIC=enviro-event
        export REGISTRY_ID=enviro-registry
        export REGION=europe-west1
        export DEVICE_ID=enviro-board
        export DATASET=enviro_dataset
        export TABLE=sensor_data

1.  Create the Pub/Sub topic:

        gcloud pubsub topics create $EVENT_TOPIC

1.  Create the IoT Core registry:

        gcloud iot registries create $REGISTRY_ID \
          --region $REGION \
          --event-notification-config=topic=$EVENT_TOPIC

1.  Create the the sensor board identity in the newly created IoT Core registry with the public key:

        cd $HOME/community/tutorials/cloud-iot-enviro-board-workshop/
        gcloud iot devices create $DEVICE_ID \
          --region=$REGION \
          --registry=$REGISTRY_ID \
          --public-key=path=device_pub_key.pem,type=es256

## Verify the data ingestion setup

You now have the building blocks set up to ingest data from the Coral Environmental Sensor Board to GCP.

In this section, you verify the end-to-end integration between the sensor board and Pub/Sub.

### Create the event topic subscription

Messages sent from the device to IoT Core are automatically published on Pub/Sub.

Create a subscription to the Pub/Sub topic:

    gcloud pubsub subscriptions create verify-event \
    --topic=$EVENT_TOPIC

Later, you use the subscription to get the messages.

### Configure the Raspberry Pi to send sensor data to IoT Core

1.  In the Raspberry Pi shell, set your GCP project ID as an environment variable:

        export PROJECT_ID=your-gcp-project-id

1.  In the Raspberry Pi shell, set your GCP project ID in the device application configuration file with this command:

        envsubst < $HOME/enviro-board/cloud_config_template.ini > $HOME/enviro-board/cloud_config.ini

### Download the CA certificate

Run the following in the Raspeberry Pi shell to download the Google root CA certificate:

    cd $HOME/enviro-board/
    wget https://pki.goog/roots.pem

The CA certificate is used to establish the chain of trust to communicate with IoT Core using the Transport Layer 
Security (TLS) protocol.

### Run the streaming script

The streaming script reads the sensor measurement values of the sensor board and publishes the data to Pub/Sub 
through IoT Core.

1.  Start the script with the following commands in the Raspberry Pi shell:

        cd $HOME/enviro-board/
        python3 enviro_demo.py --upload_delay 10
        
1.  Let the script run for 20 seconds.
1.  Press `Ctrl+C` to stop the script.

### Verify sensor data in Pub/Sub

The sensor values sent from the Raspberry Pi are published to Pub/Pub. In this step, you pull the messages from 
the Pub/Sub subscription created in a previous step and verify that the values are delivered to Pub/Sub.

1.  Run the following command in Cloud Shell to pull messages from the Pub/Sub subscription:

        gcloud pubsub subscriptions pull verify-event --auto-ack

1.  Verify that you get the messages from the Coral Environmental Sensor Board.

## Set up the Cloud Function to process sensor data

In this section, you deploy the Cloud Function that gets triggered by the sensor data messages published to Pub/Sub.
The function parses the message and adds the values to BigQuery.

In Cloud Shell, run the following:

    cd $HOME/community/tutorials/cloud-iot-enviro-board-workshop/functions
    gcloud functions deploy enviro \
     --set-env-vars=DATASET=${DATASET},TABLE=${TABLE} \
     --region ${REGION} \
     --trigger-topic ${EVENT_TOPIC} \
     --runtime nodejs8 \
     --memory 128mb

## Set up data storage

In this section, you create the data set and table in BigQuery to store the sensor data.

In Cloud Shell, run the following:

    bq mk $DATASET

    bq mk ${DATASET}.${TABLE} $HOME/community/tutorials/cloud-iot-enviro-board-workshop/bq/schema.json

## Start the sensor data stream

In this section, you restart the demonstration script on the Raspberry Pi to generate sensor data that triggers the Cloud
Function to store the data in BigQuery. You can control the interval at which the sensor data is sent to GCP by setting the
`upload_delay` parameter.

In the Raspberry Pi shell, run the following:

    cd $HOME/enviro-board/
    python3 enviro_demo.py --upload_delay 15

## View sensor data in BigQuery

1.  Open the [BigQuery console](http://console.cloud.google.com/bigquery).
1.  Paste the following query into the **Query editor**:

        SELECT * FROM `[PROJECT_ID].enviro_dataset.sensor_data`
        ORDER BY time DESC
        LIMIT 20
    
    Replace the placeholder `[PROJECT_ID]` with your project ID.
     
1.  Click **Run**.
1.  Verify that a table with sensor data is returned.
1.  Press `Ctrl+C` to stop the sensor data streaming from the Raspberry Pi shell. 

## Perform data analytics and device control

In this section, you start from the sample spreadsheet and configure it to retrieve data from your cloud project and send 
commands to your device.

### Set up Google Sheets

1.  Open the [sample spreadsheet](https://docs.google.com/spreadsheets/d/1LI07utVbiuonjZfn2ORWmC0kC5_OYc7CYw3vZF1aEuo).
1.  Click **File** > **Make a copy**.
1.  Enter a name for the copy, and click **OK**.

#### Configure OAuth2

Create an OAuth client and configure your spreadsheet to use the client:

1.  In Google Sheets, select **OAuth Configuration** in the **IoT** menu.

    This opens the **Oauth credential configuration** dialog box.
1.  Copy the authorized redirect URI shown in the **Oauth credential configuration** dialog box.
1.  Go to the [**Credentials** page](https://console.cloud.google.com/apis/credentials).
1.  Click **Create credentials**, and choose **OAuth client ID**.
1.  In the **Application type** section, choose **Web application**.
1.  Give the credential an identifiable name.
1.  Paste the authorized redirect URI that you retrieved in the second step into the **Authorized redirect URIs** field.
1.  Click **Create**.
1.  Make a copy of the client ID and the client secret that are shown after the client is created.
1.  Go back to your spreadsheet and paste the client ID and the client secret values into their corresponding fields in the
    **Oauth credential configuration** dialog box.
1.  Click **Save**, and wait for the `config saved!` response.
1.  Click **Close**.

#### Configure IoT Core

In this section, you configure your spreadsheet with your GCP project-specific values. The spreadsheet needs these values to
retrieve device data from IoT Core.

1.  In Google Sheets, select **IoT Core Configuration** in the **IoT** menu.
1.  Fill in values of your Google Cloud project, IoT Core region and registry.
1.  Click **Save**, and wait for the `config saved!` response.
1.  Click **Close**.

### Load devices

In this section, you go through the authorization steps that enable the spreadsheet to retrieve information from GCP, and
you use the spreadsheet to fetch your device ID from IoT Core.

1.  In Google Sheets, select **Load Devices** in the **IoT** menu.

    The first time you use this command, a sidebar opens.
1.  Click **Authorize** in the sidebar and continue with the authorization to allow the script to access the
    IoT Core API on your behalf.
1.  When the OAuth2 flow is successful, a new tab opens with this message: `Success! You can close this tab.`
1.  Close the tab.
1.  Select **Load Devices** in the **IoT** menu again, and verify that your device name appears in the **Devices** column.
1.  Click the checkbox next to the device name cell to select it.

### Set up the BigQuery connector

In this section, you configure the connector between Google Sheets and BigQuery to load data from your `sensor_data` table
in BigQuery.

1.  In Google Sheets, choose **Data** > **Data connectors** > **Connect to BigQuery**.
1.  Choose your GCP project from the list and click **Write query**.
1.  Paste the following query into the BigQuery query editor, replacing the placeholder `[PROJECT_ID]` with your project ID:

        SELECT * FROM `[PROJECT_ID].enviro_dataset.sensor_data`
        WHERE time > TIMESTAMP(CURRENT_DATE())
        AND device_id = @DEVICE_ID
        ORDER BY time

1.  Add a new parameter:
    1.  Click **Parameters** > **Add**.
    1.  Fill in `DEVICE_ID` for **Name** and `Sheet1!C2` for **Cell reference**.
    1.  Click **Add**.
1. Click **Insert result**.

### Send commands to your board

In this section, you use your spreadsheet to send arbitrary string messages to your Raspberry Pi, and the messages appear
on the OLED display of the Coral Environmental Sensor Board.

1.  In the cell under **Command**, write a simple text message.
1.  Choose **IoT** > **Send command to device**.
1.  Verify that you get a `Device is not connected` error message when the demo script is turned off on your Raspberry Pi.
1.  Restart the demo script from your Raspberry Pi shell by running the following: `python3 enviro_demo.py`
1.  Click **IoT** > **Send command to device** to resend the message.
1.  Verify that the message appears on the OLED display.

### Analytics in Google Sheets

Here are some things that you can do to further experiment with the data and device command functions from within your
spreadsheet:

-   Create time series graphs over the sensor data.
-   Derive moving time average values from raw sensor data.
-   Create recurring jobs that [auto-load](https://gsuiteupdates.googleblog.com/2019/02/refresh-bigquery-data-sheets.html) 
    the data from BigQuery.
-   Create function that monitors the sensor values and automatically sends commands when configured threshold values are 
    reached or exceeded.

## Cleaning up

To avoid incurring charges to your Google Cloud account for the resources used in this tutorial, you can delete
the project.

The easiest way to eliminate billing is to delete the project you created for the tutorial:

1.  In the GCP Console, go to the [**Projects** page](https://console.cloud.google.com/iam-admin/projects).
1.  In the project list, select the project you want to delete, and click **Delete**.
1.  In the dialog, type the project ID, and then click **Shut down** to delete the project.

## Next steps

-   Learn more about [Google Cloud IoT](https://cloud.google.com/solutions/iot/).
-   Learn more about
    [microcontrollers and real-time analytics](https://cloud.google.com/community/tutorials/ardu-pi-serial-part-1).
-   Learn more about
    [streaming into BigQuery](https://cloud.google.com/solutions/streaming-data-from-cloud-storage-into-bigquery-using-cloud-functions).
-   Learn more about
    [asynchronous patterns for Cloud Functions](https://cloud.google.com/community/tutorials/cloud-functions-async).