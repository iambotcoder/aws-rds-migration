

# 🚀 **MariaDB Migration to Amazon RDS**

---

## 📖 Overview

In this project, we set up an Amazon RDS MariaDB instance, migrated data from an EC2-hosted MariaDB database, and monitored the instance using CloudWatch metrics. This project covers the following tasks:

1. Creating an Amazon RDS MariaDB instance using the AWS CLI
2. Migrating data from an EC2-hosted MariaDB database to Amazon RDS
3. Monitoring the RDS instance through the Amazon RDS Console and CloudWatch

---

## 📑 Table of Contents  

1. [Prerequisites](#-prerequisites)  
2. [Architecture](#-architecture)  
3. [Task 1: Creating an Amazon RDS instance by using the AWS CLI](#-task-1-creating-an-amazon-rds-instance-by-using-the-aws-cli)  
4. [Task 2: Migrating application data to the Amazon RDS instance](#-task-2-migrating-application-data-to-the-amazon-rds-instance)  
5. [Task 3: Configuring the website to use the Amazon RDS instance](#-task-3-configuring-the-website-to-use-the-amazon-rds-instance)  
6. [Task 4: Monitoring the Amazon RDS Database](#-task-4-monitoring-the-amazon-rds-database)  
7. [Cleaning Up Resources](#-cleaning-up-resources)  
8. [Conclusion](#-conclusion)  
9. [References](#-references)  

---


## 🔑 Prerequisites

Before starting the project, ensure you have the following:

- **AWS CLI** installed and configured with the appropriate permissions
- **Amazon EC2 instance** running with MariaDB installed
- **Amazon RDS MariaDB instance** created using the AWS CLI
- **CloudWatch** enabled for the RDS instance
- Access to **MariaDB client** to interact with the database

---

## 🗺️ Architecture

![Screenshot 2025-02-17 154111](https://github.com/user-attachments/assets/2309f2f3-4aa8-4b07-83dc-821863e1da50)



### 🔄 Workflow:

1. Set up an Amazon RDS MariaDB instance using the AWS CLI.
2. Migrate data from a MariaDB database on an EC2 instance to the Amazon RDS MariaDB instance.
3. Use CloudWatch to monitor the RDS instance’s health and metrics.

---

## 📝 Task 1: Creating an Amazon RDS instance by using the AWS CLI


### **Task 1.1: Connecting to the CLI Host instance**
1. Open the EC2 Management Console.
2. In the navigation pane, select **Instances**.
3. Select the **CLI Host instance**.
4. Choose **Connect** and then click **Connect** on the EC2 Instance Connect tab.

### **Task 1.2: Configuring the AWS CLI**
1. Connect to the EC2 instance.
2. Run the command to configure AWS CLI:

   ```bash
   aws configure
   ```

   When prompted, enter:
   - AWS Access Key ID: `<AccessKey>`
   - AWS Secret Access Key: `<SecretKey>`
   - Default region name: `<LabRegion>`
   - Default output format: `json`

### **Task 1.3: Creating prerequisite components**

#### **1. Create the CafeDatabaseSG security group**
Run the following command to create a security group:

```bash
aws ec2 create-security-group \
--group-name CafeDatabaseSG \
--description "Security group for Cafe database" \
--vpc-id <CafeInstance VPC ID>
```

- Replace `<CafeInstance VPC ID>` with your VPC ID.
- Save the `GroupId` from the output.

#### **2. Add inbound rule to the security group**
Run the following command to allow inbound traffic on TCP port 3306:

```bash
aws ec2 authorize-security-group-ingress \
--group-id <CafeDatabaseSG Group ID> \
--protocol tcp --port 3306 \
--source-group <CafeSecurityGroup Group ID>
```

- Replace `<CafeDatabaseSG Group ID>` and `<CafeSecurityGroup Group ID>` with their respective values.

#### **3. Verify inbound rule**
Run this command to verify that the inbound rule is applied:

```bash
aws ec2 describe-security-groups \
--query "SecurityGroups[*].[GroupName,GroupId,IpPermissions]" \
--filters "Name=group-name,Values='CafeDatabaseSG'"
```

#### **4. Create CafeDB Private Subnet 1**
Run this command to create the first private subnet:

```bash
aws ec2 create-subnet \
--vpc-id <CafeInstance VPC ID> \
--cidr-block 10.200.2.0/23 \
--availability-zone <CafeInstance Availability Zone>
```

- Replace `<CafeInstance VPC ID>` and `<CafeInstance Availability Zone>` with the appropriate values.

#### **5. Create CafeDB Private Subnet 2**
Run this command to create the second private subnet:

```bash
aws ec2 create-subnet \
--vpc-id <CafeInstance VPC ID> \
--cidr-block 10.200.10.0/23 \
--availability-zone <availability-zone>
```

- Replace `<CafeInstance VPC ID>` and `<availability-zone>` with the appropriate values.

#### **6. Create CafeDB Subnet Group**
Run this command to create the DB subnet group:

```bash
aws rds create-db-subnet-group \
--db-subnet-group-name "CafeDB Subnet Group" \
--db-subnet-group-description "DB subnet group for Cafe" \
--subnet-ids <Cafe Private Subnet 1 ID> <Cafe Private Subnet 2 ID> \
--tags "Key=Name,Value= CafeDatabaseSubnetGroup"
```

- Replace `<Cafe Private Subnet 1 ID>` and `<Cafe Private Subnet 2 ID>` with the respective subnet IDs.

### **Task 1.4: Creating the Amazon RDS MariaDB instance**

Run the following command to create the RDS MariaDB instance:

```bash
aws rds create-db-instance \
--db-instance-identifier CafeDBInstance \
--engine mariadb \
--engine-version 10.5.13 \
--db-instance-class db.t3.micro \
--allocated-storage 20 \
--availability-zone <CafeInstance Availability Zone> \
--db-subnet-group-name "CafeDB Subnet Group" \
--vpc-security-group-ids <CafeDatabaseSG Group ID> \
--no-publicly-accessible \
--master-username root --master-user-password 'Re:Start!9'
```

- Replace `<CafeInstance Availability Zone>` and `<CafeDatabaseSG Group ID>` with the respective values.

### **Monitor the status of the RDS instance**
To check the status of the database, run:

```bash
aws rds describe-db-instances \
--db-instance-identifier CafeDBInstance \
--query "DBInstances[*].[Endpoint.Address,AvailabilityZone,PreferredBackupWindow,BackupRetentionPeriod,DBInstanceStatus]"
```

Repeat the command until the status shows **available**. Once available, you will find the endpoint address in the output.

### **Final output:**
- RDS Instance Database Endpoint Address: `<cafedbinstance.xxxxxxx.us-west-2.rds.amazonaws.com>`

These steps should help you create the Amazon RDS instance using AWS CLI! Let me know if you need further clarification.

---

## 📝 Task 2: Migrating application data to the Amazon RDS instance.

#### 1. **Connect to the EC2 Instance (CafeInstance)**:

   - First, you'll connect to the CafeInstance (EC2 instance) using EC2 Instance Connect. This instance will interact with the Amazon RDS instance through the MySQL protocol, as it is allowed by the associated security group (CafeDatabaseSG).

#### 2. **Create a Backup Using `mysqldump`**:
   - On the CafeInstance, use the `mysqldump` utility to create a backup of the local MySQL database (`cafe_db`).
   - The command:
     ```bash
     mysqldump --user=root --password='Re:Start!9' --databases cafe_db --add-drop-database > cafedb-backup.sql
     ```
   - This command:
     - Uses `mysqldump` to create an SQL backup.
     - The `--databases` option specifies the database to back up (`cafe_db`).
     - `--add-drop-database` ensures that a `DROP DATABASE` statement is included to remove the database before restoring.
     - `> cafedb-backup.sql` redirects the output into a `.sql` file (`cafedb-backup.sql`).

#### 3. **Review the Backup File**:
   - Open the `cafedb-backup.sql` file to verify its contents.
   - To view the file using the Linux `less` command:
     ```bash
     less cafedb-backup.sql
     ```
   - Navigate with the arrow keys, `Page Up/Down`, and press `q` to exit.

#### 4. **Restore Backup to Amazon RDS**:
   - To restore the backup file to the Amazon RDS database, use the `mysql` command. This command connects to the RDS instance and runs the SQL statements from the backup file.
   - Command format:
     ```bash
     mysql --user=root --password='Re:Start!9' --host=<RDS Instance Database Endpoint Address> < cafedb-backup.sql
     ```
   - Replace `<RDS Instance Database Endpoint Address>` with the actual endpoint address of the Amazon RDS instance.

#### 5. **Verify Data Migration**:
   - After restoring the backup, verify that the data was correctly transferred to the Amazon RDS instance.
   - Connect to the Amazon RDS instance using the `mysql` command:
     ```bash
     mysql --user=root --password='Re:Start!9' --host=<RDS Instance Database Endpoint Address> cafe_db
     ```
   - Query the `product` table to ensure the data has been restored:
     ```sql
     select * from product;
     ```
   - Ensure the data returned matches the expected rows.

#### 6. **Exit MySQL Session**:
   - After verifying the data, exit the MySQL session:
     ```bash
     exit
     ```

#### 7. **Leave the CafeInstance SSH Window Open**:
   - Keep the SSH window open for potential future use.

---

## 📝 Task 3: Configuring the website to use the Amazon RDS instance.

#### 1. **Access the AWS Systems Manager Console**:
   - In the AWS Management Console, search for and select **Systems Manager** to open the AWS Systems Manager Console.

#### 2. **Navigate to Parameter Store**:
   - In the left navigation pane, click on **Parameter Store** under **Application Management**.

#### 3. **Edit the `dbUrl` Parameter**:
   - In the **My Parameters** section, locate the `/cafe/dbUrl` parameter.
   - Click on the parameter name to view its details.
   - Select the **Edit** button to modify the value of the `dbUrl` parameter.

#### 4. **Update the Database URL**:
   - In the **Value** field, replace the current value with the endpoint address of the Amazon RDS instance that you recorded earlier.
   - This action updates the website’s configuration to point to the RDS database instead of the local database.

#### 5. **Save Changes**:
   - After updating the database URL, click on **Save changes** to save the new value.

#### 6. **Test the Website**:
   - Open a new browser window and enter the URL for the café website (e.g., **CafeInstanceURL**) that you copied earlier.
   - The homepage of the website should load successfully, confirming that the connection to the RDS instance is working.

#### 7. **Verify the Data**:
   - Click on the **Order History** tab on the website.
   - Compare the number of orders displayed on the website with the number of orders you recorded before migrating the database to Amazon RDS.
   - Both numbers should be the same, confirming that the data has been successfully migrated and is accessible from the new RDS instance.

#### 8. **Place New Orders (Optional)**:
   - To further verify the functionality, place some new orders on the website.
   - Ensure that the orders are placed successfully and that the data is being written to the Amazon RDS instance.

#### 9. **Close the Browser Tab**:
   - Once the testing is complete, you can close the browser tab.

--- 


## 📝 Task 4: Monitoring the Amazon RDS Database

#### 1. **Access the RDS Management Console**:
   - In the AWS Management Console, search for and select **RDS** to open the Amazon RDS Management Console.

#### 2. **Select the Database Instance**:
   - In the left navigation pane, choose **Databases**.
   - From the list of available database instances, select the **cafedbinstance**.

#### 3. **Monitor Performance Metrics**:
   - Once the database instance details are displayed, navigate to the **Monitoring** tab.
   - The **Monitoring** tab shows key metrics provided by Amazon RDS through CloudWatch, such as:
     - **CPUUtilization**: The percentage of CPU being used by the database instance.
     - **DatabaseConnections**: The number of active database connections.
     - **FreeStorageSpace**: The available storage space in the database.
     - **FreeableMemory**: The amount of free memory (RAM) available.
     - **WriteIOPS**: The average number of write operations per second.
     - **ReadIOPS**: The average number of read operations per second.
   - Some of these metrics may appear across multiple pages, so use the pagination options to explore more metrics.

#### 4. **Monitor Database Connections**:
   - To track the number of active database connections, observe the **DatabaseConnections** metric.
   - The graph will show the number of active connections over time.

#### 5. **Establish a Database Connection (Optional)**:
   - To create a connection to the RDS instance, use an interactive SQL session.
   - Open the terminal on your CafeInstance (EC2 instance) and enter the following command, replacing `<RDS Instance Database Endpoint Address>` with the actual endpoint address of your Amazon RDS instance:
     ```bash
     mysql --user=root --password='Re:Start!9' --host=<RDS Instance Database Endpoint Address> cafe_db
     ```
   - This command establishes a connection to the RDS instance, and the **DatabaseConnections** graph should now reflect one open connection.

#### 6. **Execute a Query**:
   - Once connected, run the following SQL query to retrieve data from the `product` table:
     ```sql
     select * from product;
     ```
   - Verify that the query returns the expected data.

#### 7. **Close the Database Connection**:
   - To close the database connection, type `exit` in the SQL session:
     ```bash
     exit
     ```
   - After closing the connection, wait for about a minute, then click **Refresh** in the **DatabaseConnections** graph.
   - The graph should show that the number of active connections has decreased to zero.

#### 8. **Explore Additional Metrics (Optional)**:
   - If time allows, explore the other available CloudWatch graphs to monitor additional performance metrics for the RDS instance.

--- 


## 🗑️ Cleaning Up Resources

To clean up all resources, delete them in the following order:

1. **RDS Instance**:
   - Go to **RDS Dashboard** → **Databases**.
   - Select your database and click **Delete**.
   
2. **EC2 Instance**:
   - Go to **EC2 Dashboard** → **Instances**.
   - Select the EC2 instance and click **Terminate**.

3. **Backup Files** (if any):
   - Delete any files from your local system or S3 bucket.

---

---

## 📸 Outputs & Screenshots  

This section provides the expected outputs and screenshots for each task to help visualize the steps and verify successful execution.  

### 🖼️ Task 1: Creating an Amazon RDS instance  

- ✅ Screenshot: RDS instance details in AWS Management Console
- 
![Screenshot 2025-02-16 225558](https://github.com/user-attachments/assets/e5277d70-0696-4fa7-9aa8-7c86fe6bcf01)

### 🖼️ Task 2: Migrating application data to Amazon RDS    

- ✅ Screenshot: Query results from the RDS instance
- 
![Screenshot 2025-02-17 163850](https://github.com/user-attachments/assets/38af7309-d956-411d-abd2-b58e50da0ad1)


### 🖼️ Task 3: Configuring the website to use Amazon RDS  

- ✅ Screenshot: Application configuration file update
- 
- ![Screenshot 2025-02-17 163840](https://github.com/user-attachments/assets/7ae765bc-6d6d-4c4c-8caa-ed481e6f2667)

- ✅ Screenshot: Web application running with RDS connection
- 
- ![Screenshot 2025-02-16 222451](https://github.com/user-attachments/assets/86fa7ecb-61c3-40b4-825a-67dc639ec6a9)


### 🖼️ Task 4: Monitoring the Amazon RDS Database  

- ✅ Screenshot: CloudWatch metrics (CPUUtilization, DatabaseConnections, FreeStorageSpace)
- 
![Screenshot 2025-02-17 163811](https://github.com/user-attachments/assets/896a5989-aabb-430e-9ff3-c5c21f465fa6)



## ✅ Conclusion

- Accessed the RDS Management Console and reviewed key database metrics.
- Monitored the **DatabaseConnections** metric to observe live connections.
- Used an interactive SQL session to test the connection and query the database.
- Learned how to use CloudWatch to monitor Amazon RDS performance and health metrics.


---

## References

This project was inspired by AWS RDS migration tutorials and documentation.

---
