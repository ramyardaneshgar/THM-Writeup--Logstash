# THM-Writeup--Logstash - Logstash
Writeup for TryHackMe Logstash Lab - Advanced log ingestion, parsing, and transformation techniques using Logstash for threat analysis in SOC environments.

By Ramyar Daneshgar 

## **Objective**
In this write-up, I will detail my approach to completing the *Logstash: Data Processing Unit* lab. This module focuses on leveraging Logstash—a critical component of the Elastic Stack (ELK)—to process and analyze security-relevant data, such as authentication logs and attack indicators. The tasks involved understanding Logstash configurations, writing custom pipelines, and applying advanced transformations to structured and unstructured data.

---

## **Environment Setup**
The lab environment involved setting up and verifying Logstash alongside Elasticsearch and Kibana. The following steps ensured proper installation and configuration:

### **Installing Logstash**
I began by installing Logstash using the provided `.deb` package. This was executed with:
```bash
dpkg -i logstash.deb
```
After installation, I enabled and started the service for persistence across reboots:
```bash
systemctl daemon-reload
systemctl enable logstash.service
systemctl start logstash.service
```

### **Validation**
To confirm Logstash was running, I verified its status:
```bash
systemctl status logstash.service
```
The output confirmed an active service, ensuring readiness for pipeline configurations.

---

## **Pipeline Fundamentals**
Logstash pipelines are composed of three essential components:
1. **Input**: Specifies the data source, such as files, TCP streams, or message queues.
2. **Filter**: Applies transformations, enrichments, or parsing logic to the ingested data.
3. **Output**: Sends the processed data to a destination, such as Elasticsearch, files, or syslog servers.

Each pipeline was written in the configuration directory at `/etc/logstash/conf.d/`.

---

## **Use Cases and Configurations**
### **1. Basic Stdin/Stdout Testing**
For an initial understanding of pipeline behavior, I created a simple test pipeline. This configuration used the `stdin` plugin as the input and `stdout` for the output to display results in the terminal:
```bash
/usr/share/logstash/bin/logstash -e 'input { stdin {} } output { stdout {} }'
```

When I typed "HELLO" and "LOGSTASH TEST," Logstash returned both inputs as structured events in JSON format. This validated the pipeline's basic functionality and demonstrated Logstash’s ability to handle unstructured inputs effectively.

---

### **2. Authentication Log Parsing**
Next, I created a pipeline to process Linux authentication logs (`/var/log/auth.log`) and extract timestamps for security event correlation:
```bash
input {
  file {
    path => "/var/log/auth.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  if [message] =~ /^(\w{3}\s+\d{1,2}\s+\d{2}:\d{2}:\d{2})/ {
    date {
      match => [ "message", "MMM d HH:mm:ss", "MMM dd HH:mm:ss" ]
      target => "@timestamp"
    }
  }
}

output {
  file {
    path => "/home/Desktop/logstash_output.log"
  }
}
```

#### **Logic**
1. **Input**: Read logs from `/var/log/auth.log`.
2. **Filter**: Used the `date` plugin to normalize timestamps for accurate event sequencing.
3. **Output**: Saved processed events to `logstash_output.log` for forensic analysis.

#### **Key Takeaway**
Standardizing timestamps using the `date` plugin is essential for chronological event reconstruction in incident response.

---

### **3. Parsing Web Attack Logs**
To simulate log analysis for detecting potential threats, I processed a CSV file (`web_attacks.csv`) containing mock web attack data:
```bash
input {
  file {
    path => "/home/Desktop/web_attacks.csv"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}

filter {
  csv {
    separator => ","
    columns => ["timestamp", "ip_address", "request", "referrer", "user_agent", "attack_type"]
  }

  if [attack_type] =~ /SQL Injection|Brute Force/ {
    # Flag specific attack types for further analysis
  }
}

output {
  file {
    path => "/home/Desktop/updated-web-attacks.csv"
  }
}
```

#### **Logic**
1. **Input**: Parsed the CSV file containing attack metadata.
2. **Filter**: Leveraged the `csv` plugin to extract fields and added conditional logic to flag critical attack types (e.g., SQL Injection).
3. **Output**: Stored the processed data in `updated-web-attacks.csv` for future investigation.

#### **Lesson Learned**
The `csv` plugin is particularly useful for parsing structured attack data, enabling quick identification of malicious patterns.

---

### **4. Renaming and Dropping Fields**
To optimize event data for security monitoring, I utilized the `mutate` plugin for renaming fields and dropping unnecessary logs:
```bash
filter {
  mutate {
    rename => { "src_ip" => "source_ip" }
  }

  if [status] == "error" {
    drop {}
  }
}
```

#### **Objective**
- **Renaming**: Standardized the `src_ip` field to `source_ip` to maintain consistency across multiple datasets.
- **Dropping**: Eliminated noise by removing events where `status` was `error`.

#### **Result**
This configuration ensured only actionable and normalized data was sent downstream, reducing storage and processing overhead.

---

## **Advanced Techniques**
### **Conditional Field Manipulation**
Using the `prune` filter, I removed all empty fields except specific whitelisted fields:
```bash
filter {
  prune {
    whitelist_names => ["timestamp", "attack_type"]
  }
}
```

This approach minimized the volume of irrelevant data and highlighted critical fields for analysis.

---

### **Pipeline Debugging**
To debug configurations, I used the `stdout` plugin to print events to the console:
```bash
output {
  stdout { codec => rubydebug }
}
```

The `rubydebug` codec provided detailed insights into event structures, allowing me to fine-tune the pipeline for optimal performance.

---

## **Commands and Tools**
### **Logstash Plugins**
1. **Input**:
   - `file`: Reads data from files.
   - `stdin`: Accepts terminal input.
2. **Filter**:
   - `grok`: Parses unstructured logs using regex patterns.
   - `csv`: Extracts fields from CSV-formatted data.
   - `mutate`: Renames, converts, or removes fields.
   - `prune`: Removes unwanted fields.
3. **Output**:
   - `file`: Writes events to files.
   - `stdout`: Prints events for debugging.

### **Commands**
- `logstash -f <file>`: Executes a pipeline configuration file.
- `systemctl`: Manages Logstash service states.

---

## **Lessons Learned**
1. **Field Normalization**: Renaming and standardizing fields, such as `src_ip` to `source_ip`, is critical for data consistency in SIEM systems.
2. **Data Filtering**: Leveraging conditional filters like `drop` and `prune` ensures pipelines focus only on security-relevant events.
3. **Parsing Structured Data**: The `csv` and `grok` plugins simplify the ingestion of structured and semi-structured data formats.
4. **Log Transformation**: The `mutate` plugin provides extensive flexibility in reshaping log data for downstream analysis.

---

## **Conclusion**
This lab highlighted the signifincance of Logstash as a log processing engine. From parsing unstructured authentication logs to normalizing structured attack data, I developed a deepepend understanding of d building scalable SIEM pipelines and improving log ingestion workflows in SOC environments.

