# aws-networking-practice

This project sets up an API using Flask and configures it to be accessible through Nginx. The API is connected to a private database to fetch and return data from a table.

# Architecture

![412203392-fe2739c6-6e62-4a0a-81a6-65e8dff99ca3](https://github.com/user-attachments/assets/7f3fb58e-bf47-4340-859a-9ceb0a55055e)

## Technologies Used
**Flask**: Python web framework.

**Nginx**: Web server.

**Python**: Programming language used for the API.

**RDS MySQL instance**: Database system to store and query data.

**AWS EC2**: Cloud infrastructure to deploy the public and private instances.

# Implementation steps:

## Prerequisites
1. EC2 Instances in AWS (public and private) configured in the same VPC.
2. Python 3.x and pip installed on the private instance.
3. Nginx installed on the public and private instances.


## 1. Public Instance Setup
Install Nginx:

```bash
sudo apt update
sudo apt install nginx
```
Configure Nginx as Reverse Proxy: Edit the Nginx configuration file to forward requests to the Flask API (on the private instance).

```bash
sudo vi /etc/nginx/sites-available/default
```
Content of the default file:

```nginx
server {
    listen 80;
    server_name _;

    location / {
        root /var/www/html;
        index index.html;
    }

    location /api/ {
        proxy_pass http://<privateip>:8080;  # Private IP of the private instance
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```
Create the HTML Page: Create the index.html file, which will make a request to the API.

```bash
sudo vi /var/www/html/index.html
```
Content of the index.html file:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>API via Nginx</title>
</head>
<body>
    <h1>Hello from Nginx</h1>
    <p id="message">Loading message from the API...</p>
<script> 
    fetch("http://<public_api>/api/hello")  // Use the public IP of the public instance
    .then(response => response.json())
    .then(data => {
        document.getElementById("message").innerText = data.message;
    })
    .catch(error => console.error("Error:", error));
</script>
</body>
</html>
```
Restart Nginx:


``` bash
sudo systemctl restart nginx
```

## 2. Private Instance Setup
Install Python and Dependencies: Make sure you have Python and the necessary libraries installed:

```bash

sudo apt update
sudo apt install python3-pip
pip install flask flask-cors mysql-connector-python
```
### Create and Configure the Flask API: Create the main.py file, which defines the API and queries the database.

Content of the main.py file:

```python
from flask import Flask, jsonify
from flask_cors import CORS
import mysql.connector  # Or psycopg2 for PostgreSQL

app = Flask(__name__)
CORS(app)  # Enables CORS to allow requests from the browser

# Database configuration (adjust parameters)
db_config = {
    'host': <endpoint>,  # Private IP of the database
    'user': 'username',
    'password': 'password',
    'database': 'database_name'
}

def get_data():
    conn = mysql.connector.connect(**db_config)
    cursor = conn.cursor(dictionary=True)
    cursor.execute("SELECT * FROM your_table LIMIT 10;")
    results = cursor.fetchall()
    cursor.close()
    conn.close()
    return results

@app.route('/api/data', methods=['GET'])
def fetch_table():
    try:
        data = get_data()
        return jsonify({'data': data}), 200
    except Exception as e:
        return jsonify({'error': str(e)}), 500

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)
```

```bash
nohup python3 main.py > flask.log 2>&1 &
```

## 3. Verify the Connection and API
Open the browser or use curl to access the homepage of the public instance:


To fetch data from the database, access the /api/data endpoint:

```bash

curl http://<public_ip>/api/data
```

## How It Works
1. Public Instance: Receives HTTP requests from the client (browser) and forwards the requests to the private instance running the Flask API.
Nginx as Reverse Proxy: Nginx acts as an intermediary between the client and the Flask API server, ensuring that the API is hidden behind the private network.
2. Flask API: The API is running on the private instance and connects to a database (MySQL) to return results from a table when a request is made to the corresponding endpoint.

### Security Considerations
Ensure that the security group rules in AWS are configured to allow traffic from the public instance to the private instance on the appropriate port (e.g., port 8080 for the API).
Never expose the API directly to the internet without a reverse proxy like Nginx to ensure security.
