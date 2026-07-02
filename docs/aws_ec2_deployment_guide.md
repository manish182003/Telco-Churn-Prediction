# AWS EC2 Free-Tier Deployment & CI/CD Setup Guide

This guide details the step-by-step process of launching an AWS EC2 instance, configuring it under the AWS Free Tier, installing Docker, and setting up the GitHub Actions CI/CD secrets for automated deployment.

---

## Step 1: Launch an AWS EC2 Free-Tier Instance

1. Log in to your [AWS Management Console](https://aws.amazon.com/console/).
2. Navigate to the **EC2 Console** and click **Launch Instance**.
3. Configure the following settings:
   * **Name**: `telco-churn-serving-app` (or any name you prefer)
   * **Application and OS Images (AMI)**: Choose **Ubuntu** (specifically **Ubuntu Server 22.04 LTS**, which is Free Tier eligible).
   * **Instance Type**: Select **`t2.micro`** (or **`t3.micro`** if you are in a region where it is the Free Tier default).
   * **Key Pair**: Select an existing key pair or click **Create new key pair**. Download the `.pem` file (e.g., `telco-key.pem`) and keep it secure.
   * **Network Settings**:
     * Select **Create security group**.
     * Check **Allow SSH traffic from** (it is best practice to limit this to your IP, but `Anywhere 0.0.0.0/0` works for testing).
     * Check **Allow HTTP traffic from the internet** (optional, but we will add a custom rule for port `8000` next).
4. Click **Launch Instance**.

---

## Step 2: Configure EC2 Security Group for Port 8000

Our FastAPI and Gradio application runs on port `8000`. We need to allow inbound traffic on this port.

1. In the EC2 Console, go to **Instances** and click on your running instance.
2. Select the **Security** tab at the bottom, and click on the link under **Security groups** (e.g., `sg-xxxxxxxx`).
3. Click **Edit inbound rules**.
4. Click **Add rule** and set:
   * **Type**: `Custom TCP`
   * **Port Range**: `8000`
   * **Source**: `Anywhere-IPv4` (`0.0.0.0/0`)
   * **Description**: `FastAPI / Gradio serving port`
5. Click **Save rules**.

---

## Step 3: Install Docker on the EC2 Instance

1. Open your local terminal (Git Bash, Command Prompt, or PowerShell) and SSH into your EC2 instance:
   ```bash
   ssh -i /path/to/telco-key.pem ubuntu@<your-ec2-public-ip>
   ```
2. Run the following commands to update Ubuntu and install Docker:
   ```bash
   # Update apt package index
   sudo apt-get update -y
   
   # Install Docker
   sudo apt-get install -y docker.io
   
   # Start and enable Docker service
   sudo systemctl start docker
   sudo systemctl enable docker
   
   # Add ubuntu user to the docker group so you don't need 'sudo' for docker commands
   sudo usermod -aG docker ubuntu
   ```
3. **Exit and Reconnect**: Type `exit` and then SSH back in. This ensures the docker group membership takes effect:
   ```bash
   ssh -i /path/to/telco-key.pem ubuntu@<your-ec2-public-ip>
   
   # Verify docker runs without sudo
   docker ps
   ```

---

## Step 4: Create a Free Docker Hub Repository

GitHub Actions will build your Docker image and push it to Docker Hub, which your EC2 instance will then pull from.

1. Go to [Docker Hub](https://hub.docker.com/) and log in (or create a free account).
2. Click **Create Repository**.
3. Set the repository name to: **`telco-churn-app`**.
4. Leave it as **Public** and click **Create**.
5. Create an Access Token:
   * Click your profile icon at the top right -> **Account Settings** -> **Security**.
   * Click **New Access Token**.
   * Name it `github-actions-token`, select read/write permissions, and click **Generate**.
   * **Copy the token value**; you will need it in the next step.

---

## Step 5: Configure GitHub Secrets

For your GitHub Actions workflow to securely log in to Docker Hub and SSH into your EC2 instance, you need to save your credentials as repository secrets:

1. Open your repository on GitHub.
2. Go to **Settings** -> **Secrets and variables** -> **Actions**.
3. Click **New repository secret** and add the following 5 secrets:

| Secret Name | Value Description | Example Value |
| :--- | :--- | :--- |
| **`DOCKERHUB_USERNAME`** | Your Docker Hub account username. | `johndoe` |
| **`DOCKERHUB_TOKEN`** | The Personal Access Token generated in Step 4. | `dckr_pat_...` |
| **`EC2_HOST`** | The **Public IP Address** (or Public IPv4 DNS) of your EC2 instance. | `54.210.34.12` |
| **`EC2_USERNAME`** | The default SSH login username for the EC2 OS. | `ubuntu` |
| **`EC2_SSH_KEY`** | The **entire contents** of your private SSH key (`.pem` file). | `-----BEGIN RSA PRIVATE KEY----- ...` |

> [!TIP]
> To copy the exact value of your `.pem` key on Windows:
> Open the file in Notepad, copy everything (including `-----BEGIN RSA PRIVATE KEY-----` and `-----END RSA PRIVATE KEY-----`), and paste it into the GitHub secret input.

---

## Step 6: Commit and Push to Trigger CI/CD

Since you reset your git history, run these commands to push the project code to your remote GitHub repository for the first time:

```bash
# Add all files
git add .

# Create the first commit
git commit -m "feat: initial commit with end-to-end ML pipeline, docker serving, and CI/CD"

# Rename branch to main
git branch -M main

# Link to your remote GitHub repository
git remote add origin <your-github-repo-url>

# Force push to set up the upstream branch and trigger the GitHub Actions deploy
git push -u origin main --force
```

Once pushed, go to the **Actions** tab of your repository on GitHub to watch the workflow build your Docker container, push it to Docker Hub, SSH into your EC2 instance, and deploy it!

Once the deployment completes, navigate to `http://<your-ec2-public-ip>:8000/ui` in your web browser to test your live, productionized machine learning model interface!
