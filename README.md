# **Automated Web & Database Server Deployment Guide**

## **Overview**

This guide provides a step-by-step explanation of an automated script that sets up a **MariaDB database** and an **Apache web server** for an e-commerce application. The script ensures all necessary dependencies are installed, configures firewall rules, and deploys the application.

## **Prerequisites**

- A Linux server (tested on **RHEL/CentOS 7/8**)
- `sudo` privileges
- Internet access to install packages and clone repositories

## **Script Breakdown**

### **1. Print Messages in Color**

The `print_color_message` function prints colored messages in green (success) and red (failure).

```bash
print_color_message() {
  local NC="\033[0m" # No Color
  local Color
  case "$1" in
    "green") Color="\033[0;32m" ;;
    "red") Color="\033[0;31m" ;;
    *) Color="\033[0m" ;;
  esac
  echo -e "${Color}$2${NC}"
}
```

### **2. Checking If a Service is Active**

This function checks if a given service is running. If not, it exits with an error.

```bash
check_service() {
  local service="$1"
  if systemctl is-active --quiet "$service"; then
    print_color_message "green" "$service is running."
  else
    print_color_message "red" "$service is not running."
    exit 1
  fi
}
```

### **3. Installing Required Packages**

Installs a package if it is not already installed.

```bash
install_package() {
  local package="$1"
  if ! rpm -q "$package" &> /dev/null; then
    print_color_message "green" "Installing $package..."
    sudo yum install -y "$package"
  else
    print_color_message "red" "$package is already installed."
  fi
}
```

### **4. Configuring Firewall Rules**

Ensures required ports are open.

```bash
ensure_firewalld_rules() {
  local port="$1"
  if sudo firewall-cmd --list-ports | grep -q "$port/tcp"; then
    print_color_message "green" "Port $port is already open."
  else
    sudo firewall-cmd --add-port="$port/tcp" --zone=public --permanent
    sudo firewall-cmd --reload
    print_color_message "green" "Opened port $port."
  fi
}
```

### **5. Setting Up the Database**

```bash
setup_database() {
  print_color_message "green" "Setting up Database Server..."
  install_package firewalld
  sudo systemctl start firewalld
  sudo systemctl enable firewalld
  check_service "firewalld"

  install_package "mariadb-server"
  sudo systemctl start mariadb  
  sudo systemctl enable mariadb
  check_service "mariadb"
  ensure_firewalld_rules "3306"

  cat > setup-db.sql <<EOF
CREATE DATABASE IF NOT EXISTS ecomdb;
CREATE USER IF NOT EXISTS 'ecomuser'@'localhost' IDENTIFIED BY 'ecompassword';
GRANT ALL PRIVILEGES ON ecomdb.* TO 'ecomuser'@'localhost';
FLUSH PRIVILEGES;
EOF

  sudo mysql < setup-db.sql
  print_color_message "green" "Database setup completed."
}
```

### **6. Setting Up the Web Server**

```bash
setup_web_server() {
  print_color_message "green" "Setting up Web Server..."
  install_package "httpd"
  install_package "php"
  install_package "php-mysqlnd"
  install_package "git"

  sudo systemctl start httpd
  sudo systemctl enable httpd
  check_service "httpd"
  ensure_firewalld_rules "80"

  sudo git clone https://github.com/kodekloudhub/learning-app-ecommerce.git /var/www/html/
  sudo sed -i 's/172.20.1.101/localhost/g' /var/www/html/index.php
  print_color_message "green" "Web Server setup completed."
}
```

### **7. Testing Webpage Accessibility**

```bash
test_web_webpage() {
  local test_web
  test_web=$(curl -s http://localhost)
  for item in Laptop Drone VR Watch Phone; do
    if [[ $test_web == *"item"* ]]; then
      print_color_message "green" "Webpage Test Successful."
    else
      print_color_message "red" "Webpage Test Failed."
      exit 1
    fi
  done
  print_color_message "green" "Webpage Test Successful."
}
```

### **8. Final Message**

```bash
print_color_message "green" "Deployment completed Successfully"
```

## **How to Use This Script**

### **Step 1: Clone the Repository**

```bash
git clone https://github.com/rman90/Linux_Projects.git
cd ~/rman90/Linux_Projects.git
```

### **Step 2: Run the Script**

Make the script executable:

```bash
chmod +x deploy.sh
```

Run the script with:

```bash
sudo ./deploy.sh
```

## **Expected Output**

- **MariaDB** is installed and running.
- **Apache Web Server** is installed and running.
- **Firewall ports 80 and 3306** are open.
- The **e-commerce application** is deployed.
- A success message:
  ```bash
  Deployment completed Successfully
  ```

## **Troubleshooting**

- Check service statuses:
  ```bash
  systemctl status firewalld mariadb httpd
  ```
- Verify firewall rules:
  ```bash
  sudo firewall-cmd --list-ports
  ```
- Check database connection:
  ```bash
  mysql -u ecomuser -p -h localhost -e "SHOW DATABASES;"
  ```

## **License**

This project is open-source and free to use. Feel free to modify and improve it!

