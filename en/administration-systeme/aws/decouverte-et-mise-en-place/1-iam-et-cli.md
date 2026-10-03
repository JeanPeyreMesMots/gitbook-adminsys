# 1 - IAM & CLI

Let's now install the AWS CLI on an Ubuntu VM, create an IAM user (with an access key), then configure the CLI with it (`aws configure`).&#x20;

### 1. Installing the AWS CLI

Done on an Ubuntu VM, following the [official installation guide](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html).

Once configured, the CLI uses a dedicated configuration folder:

```bash
jpmm@kos-boss:~/.aws$ ll
total 16
drwxrwxr-x  2 jpmm jpmm 4096 mai   29 20:03 ./
drwxr-x--- 22 jpmm jpmm 4096 mai   29 20:03 ../
-rw-------  1 jpmm jpmm   28 mai   29 20:03 config
-rw-------  1 jpmm jpmm  116 mai   29 20:03 credentials
```

* `config` holds the general settings (region, output format, etc.).
* `credentials` holds the access credentials (access key / secret key).

[Command completion](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-completion.html) can also be enabled to make the CLI easier to use.

### 2. Creating an IAM user (console demo)

In the AWS console:

1. Go to **IAM** → **Users**.
2. Click **Add user**.
3. Enable **console access** for this IAM user.
4. Set a **custom password**, without requiring a reset at first sign-in.
5. **Attach existing policies** as needed. No tags at this step.
6. Go back to the user list, select the new user, then open the **Security credentials** tab.
7. In the CLI section, acknowledge the recommendation (**"CLI, I understand"**).
8. **Create access key**, which will be used to configure the AWS CLI.

### 3. Configuring the AWS CLI

Once the access key is generated, configure the CLI with:

```bash
aws configure
```

It asks for:

* The **Access key ID** and **Secret access key** from the previous step.
* The **default region**: an important setting, since it is used by every CLI command that doesn't explicitly specify another one. I kept **us-east-1**, the region used in Cocadmin's course.
* The **output format**: `json` by default.

The keys are saved in `~/.aws/credentials`, and the region and output format in `~/.aws/config`. One last tip: switch the AWS console to English, since most tutorials and documentation are in English.

### 4. First test command

```bash
aws s3 ls
```

This command lists the S3 buckets accessible with the configured credentials, which confirms the CLI is set up correctly.