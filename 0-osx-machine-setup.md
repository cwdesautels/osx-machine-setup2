# Machine Setup

## OSX
1. Install XCode Command Line Tools:

    ~~~
    xcode-select --install
    ~~~

2. Install Brew Package Manager:

    ~~~
    # reference: https://brew.sh:
    /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
    ~~~

3. Install required packages:

    ~~~
    brew install git kubectl helm awscli asdf helmsman terragrunt sops maven podman java terraform python pipx
    brew install --cask intellij-idea postman visual-studio-code
    ~~~

4. Install rosetta2 amd64 instruction emulation:

    ~~~
    softwareupdate --install-rosetta --agree-to-license
    ~~~

5. Set blazingly fast keyrepeat, after applying restart:

    ~~~
    defaults write -g KeyRepeat -int 1
    defaults write -g InitialKeyRepeat -int 10
    defaults write -g ApplePressAndHoldEnabled -bool false
    ~~~

6. (Optional) Get SF Mono font: https://developer.apple.com/fonts/ 

## Java
1. (Optional) Manage JDK versions via brew:

    ~~~
    brew install openjdk@11 openjdk@8 openjdk@17
    # install location, You will need to edit your profile (eg: ~/.zprofile) to create the JAVA_HOME environment variable
    #/usr/local/opt/openjdk@8
    #/usr/local/opt/openjdk@11
    #/usr/local/opt/openjdk@17
    ~~~

2. (Optional) Manage JDK versions via asdf:

    ~~~
    asdf plugin add java
    asdf list all java
    # These versions serve as examples
    asdf install java corretto-8.332.08.1 
    asdf install java corretto-11.0.15.9.1
    asdf install java corretto-17.0.3.6.1
    # install location
    #~/.asdf/installs/java/
    # Toggle between versions, this will set $JAVA_HOME
    #asdf global java corretto-8.332.08.1
    #asdf global java corretto-11.0.15.9.1
    #asdf global java corretto-17.0.3.6.1
    # Set JAVA_HOME
    #. ~/.asdf/plugins/java/set-java-home.zsh
    ~~~

3. Add certifcates to your JDK(s):

    ~~~
    # Download cert yourself (https://stackoverflow.com/a/65435907) from Nexus, ask IT for the Nexus url and you will need to be on the VPN
    # JDK 8+
    ${JAVA_HOME}/bin/keytool -import -trustcacerts -cacerts -storepass changeit -noprompt -alias czrs-nexus -file [private_cert]
    # JDK 8
    ${JAVA_HOME}/bin/keytool -import -keystore ${JAVA_HOME}/jre/lib/security/cacerts -storepass changeit -noprompt -alias czrs-nexus -file [private_cert]
    # IntelliJ
    /Applications/IntelliJ\ IDEA.app/Contents/jbr/Contents/Home/bin/keytool -import -trustcacerts -cacerts -storepass changeit -noprompt -alias czrs-nexus -file [private_cert]
    ~~~

## Git
1. Configure Git user, change John Doe to your user:

    ~~~
    git config --global user.name "John Doe"
    git config --global user.email johndoe@example.com
    ~~~

## SSH
1. Create SSH keys for your machine using the tutorial [here](https://docs.gitlab.com/ee/user/ssh.html):

    ~~~
    # You will need to edit the ssh config file to specify your created ssh key
    mkdir -p ~/.ssh
    curl https://gitlab.com/-/snippets/2592790/raw/main/2-ssh-config -o ~/.ssh/config
    ~~~

2. Upload your SSH public key to your Gitlab profile [here](https://gitlab.com/-/profile/keys).

3. Add your created SSH key to OSX keychain:

    ~~~
    ssh-add --apple-use-keychain --apple-load-keychain ~/.ssh/[private_key]
    ~~~

## Maven
1. Update Maven settings.xml:

    ~~~
    # You will need to edit your profile (eg: ~/.zprofile) to create the NEXUS_URL environment variable
    mkdir -p ~/.m2
    curl https://gitlab.com/-/snippets/2592790/raw/main/1-czrs-settings.xml -o ~/.m2/settings.xml
    # When executing maven you can specify the settings.xml to use
    mvn clean install --settings ~/.m2/settings.xml
    # Add some maven optimizations, in later IntelliJ version's Maven settings you can utilize these files as well
    mkdir -p ~/.mvn
    curl https://gitlab.com/-/snippets/2592790/raw/main/3-maven-jvm.config -o ~/.mvn/jvm.config
    curl https://gitlab.com/-/snippets/2592790/raw/main/4-maven-maven.config -o ~/.mvn/maven.config
    ~~~

## VPN
1. Ask IT, this is required for all subsequent steps 

## AWS SSO
1. Install CLI tooling:

    ~~~
    pipx ensurepath
    pipx install aws-sso-util aws-export-credentials
    ~~~

2. Configure AWS SSO, requires VPN:

    ~~~
    # Default the region and output format only, you will not have an access ID or key
    aws configure
    # Ask IT for the SSO start url
    # For each profile you want to configure (dev, nonprod, prod etc) you will need to run the following:
    aws configure sso
    # Your ~/.aws/config file should be populated now
    ~~~

3. Test AWS SSO:

    ~~~
    aws sso login --profile [profile]
    # EKS test
    aws eks update-kubeconfig --name eks03 --profile [profile]
    kubectl get pods -n [namespace]
    # Helm test
    helm list -n [namespace]
    # EKS test with export credentials
    aws-export-credentials --profile [profile] --exec kubectl get pods -n [namespace]
    ~~~

## Podman
1. Setup podman, requires VPN and AWS SSO:

    ~~~
    podman machine init --cpus=4
    podman machine start
    ~~~

2. Test Podman:

    ~~~
    # Test podman with public image
    podman run -it ubuntu:latest /bin/bash
    # Use private ECR
    aws-export-credentials --profile [profile_prod] --exec aws ecr get-login-password | podman login --username AWS --password-stdin [ecr_url]
    # Test podman with private image
    podman run -it [ecr_url]/[ecr_image] /bin/bash
    # Shutdown command
    podman machine stop
    ~~~

## Terraform
1. Setup permissions, requires AWS SSO. Create a token with *read_repository* permission and an expiration date: https://gitlab.com/-/profile/personal_access_tokens

    ~~~
    # Alternatively, you can export AWS SSO session variables in your console and forgo the following sso login and sso export
    aws sso login --profile [profile]
    # Execute this in the lowest level folder of our common terragrunt structure, for example: [project_root]/terragrunt/[aws_account]/[aws_region]/[terragrunt_module]/
    aws-export-credentials --profile [profile] --exec terragrunt init
    # When utilizing git source backed terraform modules you will be prompted for credentials
    Username for 'https://gitlab.com': [access_token_name]
    Password for 'https://[access_token_name]@gitlab.com': [access_token_key] 
    ~~~

2. Test terraform

    ~~~
    aws-export-credentials --profile [profile] --exec terragrunt plan
    ~~~

## IntelliJ
1. Configure Maven plugin to download sources:
   * Navigate to: **IntelliJ > Preferences > Build > Build Tools > Maven > Importing**, automatically download sources and documentation.
2. Configure Maven plugin to run using installed JRE:
   * Navigate to: **IntelliJ > Preferences > Build > Build Tools > Maven > Runner**, change JRE to an installed version.
3. Configure compiler plugin to enable annotation processing:
   * Navigate to: **IntelliJ > Preferences > Build > Compiler > Annotation Processors**, enable annotation processing.
4. Configure default code style:
   * Navigate to: **IntelliJ > Preferences > Code Style > Java**, change both import rules to replace with `*` to `999`.
5. Configure coding font (Optional):
   * Navigate to: **IntelliJ > Preferences > Editor > Font**, change the font to SF Mono, this was downloaded earlier in the OSX section.
