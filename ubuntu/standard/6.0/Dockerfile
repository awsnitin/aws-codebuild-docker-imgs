# Copyright 2020-2022 Amazon.com, Inc. or its affiliates. All Rights Reserved.
#
# Licensed under the Amazon Software License (the "License"). You may not use this file except in compliance with the License.
# A copy of the License is located at
#
#    http://aws.amazon.com/asl/
#
# or in the "license" file accompanying this file.
# This file is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, express or implied.
# See the License for the specific language governing permissions and limitations under the License.
FROM public.ecr.aws/ubuntu/ubuntu:22.04 AS core

ARG DEBIAN_FRONTEND="noninteractive"

# Install git, SSH, and other utilities
RUN set -ex \
    && echo 'Acquire::CompressionTypes::Order:: "gz";' > /etc/apt/apt.conf.d/99use-gzip-compression \
    && apt-get update \
    && apt install -y -qq apt-transport-https gnupg ca-certificates \
    && apt-key adv --keyserver hkp://keyserver.ubuntu.com:80 --recv-keys 3FA7E0328081BFF6A14DA29AA6A19B38D3D831EF \
    && apt-get install software-properties-common -y -qq --no-install-recommends \
    && apt-add-repository -y ppa:git-core/ppa \
    && apt-get update \
    && apt-get install git=1:2.* -y -qq --no-install-recommends \
    && git version \
    && apt-get install -y -qq --no-install-recommends openssh-client \
    && mkdir ~/.ssh \
    && mkdir -p /codebuild/image/config \
    && touch ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H github.com >> ~/.ssh/known_hosts \
    && ssh-keyscan -t rsa,dsa,ed25519,ecdsa -H bitbucket.org >> ~/.ssh/known_hosts \
    && chmod 600 ~/.ssh/known_hosts \
    && apt-get install -y -qq --no-install-recommends \
          apt-utils asciidoc autoconf automake build-essential bzip2 \
          bzr curl dirmngr docbook-xml docbook-xsl dpkg-dev \
          e2fsprogs expect fakeroot file g++ gcc gettext gettext-base \
          groff gzip iptables jq less libapr1 libaprutil1 \
          libargon2-0-dev libbz2-dev libc6-dev libcurl4-openssl-dev \
          libdb-dev libdbd-sqlite3-perl libdbi-perl libdpkg-perl \
          libedit-dev liberror-perl libevent-dev libffi-dev libgeoip-dev \
          libglib2.0-dev libhttp-date-perl libio-pty-perl libjpeg-dev \
          libkrb5-dev liblzma-dev libmagickcore-dev libmagickwand-dev \
          libmysqlclient-dev libncurses5-dev libncursesw5-dev libonig-dev \
          libpq-dev libreadline-dev libserf-1-1 libsodium-dev libsqlite3-dev libssl-dev \
          libsvn1 libsvn-perl libtcl8.6 libtidy-dev libtimedate-perl \
          libtool libwebp-dev libxml2-dev libxml2-utils libxslt1-dev \
          libyaml-dev libyaml-perl llvm locales make mlocate \
          netbase openssl patch pkg-config procps python3-configobj \
          python3-openssl rsync sgml-base sgml-data \
          tar tcl tcl8.6 tk tk-dev unzip wget xfsprogs xml-core xmlto xsltproc \
          libzip-dev vim xvfb xz-utils zip zlib1g-dev \
    && rm -rf /var/lib/apt/lists/*

ENV LC_CTYPE="C.UTF-8"

RUN useradd codebuild-user

#=======================End of layer: core  =================


FROM core AS tools

# Install stunnel
RUN set -ex \
   && STUNNEL_VERSION=5.67 \
   && STUNNEL_TAR=stunnel-$STUNNEL_VERSION.tar.gz \
   && STUNNEL_SHA256="3086939ee6407516c59b0ba3fbf555338f9d52f459bcab6337c0f00e91ea8456" \
   && curl -o $STUNNEL_TAR https://www.usenix.org.uk/mirrors/stunnel/archive/5.x/$STUNNEL_TAR && echo "$STUNNEL_SHA256 $STUNNEL_TAR" | sha256sum --check && tar xfz $STUNNEL_TAR \
   && cd stunnel-$STUNNEL_VERSION \
   && ./configure \
   && make -j4 \
   && make install \
   && openssl genrsa -out key.pem 2048 \
   && openssl req -new -x509 -key key.pem -out cert.pem -days 1095 -subj "/C=US/ST=Washington/L=Seattle/O=Amazon/OU=Codebuild/CN=codebuild.amazon.com" \
   && cat key.pem cert.pem >> /usr/local/etc/stunnel/stunnel.pem \
   && cd .. && rm -rf stunnel-${STUNNEL_VERSION}*

# AWS Tools
# https://docs.aws.amazon.com/eks/latest/userguide/install-aws-iam-authenticator.html https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_installation.html
RUN curl -sS -o /usr/local/bin/aws-iam-authenticator https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/aws-iam-authenticator \
    && curl -sS -o /usr/local/bin/kubectl https://s3.us-west-2.amazonaws.com/amazon-eks/1.22.6/2022-03-09/bin/linux/amd64/kubectl \
    && curl -sS -o /usr/local/bin/ecs-cli https://s3.amazonaws.com/amazon-ecs-cli/ecs-cli-linux-amd64-latest \
    && curl -sS -L https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_Linux_amd64.tar.gz | tar xz -C /usr/local/bin \
    && chmod +x /usr/local/bin/kubectl /usr/local/bin/aws-iam-authenticator /usr/local/bin/ecs-cli /usr/local/bin/eksctl

# Configure SSM
RUN set -ex \
    && mkdir /tmp/ssm \
    && cd /tmp/ssm \
    && wget -q https://s3.amazonaws.com/amazon-ssm-us-east-1/latest/debian_amd64/amazon-ssm-agent.deb \
    && dpkg -i amazon-ssm-agent.deb

# Install AWS CLI v2
# https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html
RUN curl https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip -o /tmp/awscliv2.zip \
    && unzip -q /tmp/awscliv2.zip -d /opt \
    && /opt/aws/install --update -i /usr/local/aws-cli -b /usr/local/bin \
    && rm /tmp/awscliv2.zip \
    && rm -rf /opt/aws \
    && aws --version

# Install env tools for runtimes
# Dotnet
ENV PATH "/root/.dotnet/:/root/.dotnet/tools/:$PATH"
RUN set -ex  \
&& wget -qO /usr/local/bin/dotnet-install.sh https://dot.net/v1/dotnet-install.sh \
&& chmod +x /usr/local/bin/dotnet-install.sh

#nodejs
ARG SRC_DIR="/usr/src"
ARG N_SRC_DIR="$SRC_DIR/n"
RUN git clone https://github.com/tj/n $N_SRC_DIR \
     && cd $N_SRC_DIR && make install 

#ruby
ARG RBENV_SRC_DIR="/usr/local/rbenv"

ENV PATH="/root/.rbenv/shims:$RBENV_SRC_DIR/bin:$RBENV_SRC_DIR/shims:$PATH" \
    RUBY_BUILD_SRC_DIR="$RBENV_SRC_DIR/plugins/ruby-build"

RUN set -ex \
    && git clone https://github.com/rbenv/rbenv.git $RBENV_SRC_DIR \
    && mkdir -p $RBENV_SRC_DIR/plugins \
    && git clone https://github.com/rbenv/ruby-build.git $RUBY_BUILD_SRC_DIR \
    && sh $RUBY_BUILD_SRC_DIR/install.sh

#python
RUN curl https://pyenv.run | bash
ENV PATH="/root/.pyenv/shims:/root/.pyenv/bin:$PATH"

#php
RUN curl -L https://raw.githubusercontent.com/phpenv/phpenv-installer/master/bin/phpenv-installer | bash
ENV PATH="/root/.phpenv/shims:/root/.phpenv/bin:$PATH"

#go
RUN git clone https://github.com/syndbg/goenv.git $HOME/.goenv
ENV PATH="/root/.goenv/shims:/root/.goenv/bin:/go/bin:$PATH"
ENV GOENV_DISABLE_GOPATH=1
ENV GOPATH="/go"

#=======================End of layer: tools  =================
FROM tools AS runtimes

#****************     .NET-CORE     *******************************************************

ENV DOTNET_6_SDK_VERSION="6.0.405"
ENV DOTNET_ROOT="/root/.dotnet"

# Add .NET Core 6 Global Tools install folder to PATH
RUN  /usr/local/bin/dotnet-install.sh -v $DOTNET_6_SDK_VERSION \
     && dotnet --list-sdks \
     && rm -rf /tmp/*

# Trigger the population of the local package cache
ENV NUGET_XMLDOC_MODE skip
RUN set -ex \
    && mkdir warmup \
    && cd warmup \
    && dotnet new \
    && cd .. \
    && rm -rf warmup \
    && rm -rf /tmp/NuGetScratch

# Install Powershell Core
# See instructions at https://docs.microsoft.com/en-us/powershell/scripting/setup/installing-powershell-core-on-linux
ARG POWERSHELL_VERSION=7.2.8
ARG POWERSHELL_DOWNLOAD_URL=https://github.com/PowerShell/PowerShell/releases/download/v$POWERSHELL_VERSION/powershell-$POWERSHELL_VERSION-linux-x64.tar.gz
ARG POWERSHELL_DOWNLOAD_SHA=28FF2653667AC63B508F0B98433E48F64E6BC1EC59F8C1D252BA89EB5A7441A2

RUN set -ex \
    && curl -SL $POWERSHELL_DOWNLOAD_URL --output powershell.tar.gz \
    && echo "$POWERSHELL_DOWNLOAD_SHA powershell.tar.gz" | sha256sum -c - \
    && mkdir -p /opt/microsoft/powershell/$POWERSHELL_VERSION \
    && tar zxf powershell.tar.gz -C /opt/microsoft/powershell/$POWERSHELL_VERSION \
    && rm powershell.tar.gz \
    && ln -s /opt/microsoft/powershell/$POWERSHELL_VERSION/pwsh /usr/bin/pwsh
#****************     END .NET-CORE     *******************************************************


#****************      NODEJS     ****************************************************

ENV NODE_16_VERSION="16.19.0"

RUN  n $NODE_16_VERSION && npm install --save-dev -g -f grunt && npm install --save-dev -g -f grunt-cli && npm install --save-dev -g -f webpack \
     && curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | apt-key add - \
     && echo "deb https://dl.yarnpkg.com/debian/ stable main" | tee /etc/apt/sources.list.d/yarn.list \
     && apt-get update && apt-get install -y -qq --no-install-recommends yarn \
     && yarn --version \
     && cd / && rm -rf $N_SRC_DIR && rm -rf /tmp/*

#****************      END NODEJS     ****************************************************

#**************** RUBY *********************************************************

ENV RUBY_31_VERSION="3.1.3"

RUN rbenv install $RUBY_31_VERSION && rm -rf /tmp/* \
    && rbenv global $RUBY_31_VERSION && ruby -v

#**************** END RUBY *****************************************************

#**************** PYTHON *****************************************************
ENV PYTHON_310_VERSION="3.10.9"

ARG PYTHON_PIP_VERSION=22.1.2
ENV PYYAML_VERSION=5.4.1

COPY tools/runtime_configs/python/$PYTHON_310_VERSION /root/.pyenv/plugins/python-build/share/python-build/$PYTHON_310_VERSION
RUN   env PYTHON_CONFIGURE_OPTS="--enable-shared" pyenv install $PYTHON_310_VERSION && rm -rf /tmp/*
RUN   pyenv global  $PYTHON_310_VERSION
RUN set -ex \
    && pip3 install --no-cache-dir --upgrade --force-reinstall "pip==$PYTHON_PIP_VERSION" \
    && pip3 install --no-cache-dir --upgrade "PyYAML==$PYYAML_VERSION" \
    && pip3 install --no-cache-dir --upgrade 'setuptools==62.6.0' wheel aws-sam-cli boto3 pipenv virtualenv

#**************** END PYTHON *****************************************************

#****************      PHP     ****************************************************
ENV PHP_81_VERSION="8.1.13"

COPY tools/runtime_configs/php/$PHP_81_VERSION /root/.phpenv/plugins/php-build/share/php-build/definitions/$PHP_81_VERSION
RUN phpenv install $PHP_81_VERSION && rm -rf /tmp/* && phpenv global $PHP_81_VERSION
RUN echo "memory_limit = 1G;" >> "/root/.phpenv/versions/$PHP_81_VERSION/etc/conf.d/memory.ini"

# Install Composer globally
RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/bin --filename=composer
#****************      END PHP     ****************************************************

#****************     GOLANG     ****************************************************
ENV GOLANG_18_VERSION="1.18.9"

RUN goenv install $GOLANG_18_VERSION && rm -rf /tmp/* && \
    goenv global  $GOLANG_18_VERSION && \
    go env -w GO111MODULE=auto

RUN go get -u github.com/golang/dep/cmd/dep
#****************      END GOLANG     *******************************

#=======================End of layer: runtimes  =================

FROM runtimes AS runtimes_n_corretto

#****************      JAVA     ****************************************************
ENV JAVA_17_HOME="/usr/lib/jvm/java-17-amazon-corretto" \
    JDK_17_HOME="/usr/lib/jvm/java-17-amazon-corretto" \
    JRE_17_HOME="/usr/lib/jvm/java-17-amazon-corretto"
ARG ANT_VERSION=1.10.12
ARG MAVEN_HOME="/opt/maven"
ARG MAVEN_VERSION=3.8.7
ARG INSTALLED_GRADLE_VERSIONS="7.6"
ARG GRADLE_VERSION=7.6
ARG SBT_VERSION=1.8.2
ARG GRADLE_PATH="$SRC_DIR/gradle"
ARG ANT_DOWNLOAD_SHA512="2287dc5cfc21043c14e5413f9afb1c87c9f266ec2a9ba2d3bf2285446f6e4ccb59b558bf2e5c57911a05dfa293c7d5c7ad60ac9f744ba11406f4e6f9a27b2403"
ARG MAVEN_DOWNLOAD_SHA512="21c2be0a180a326353e8f6d12289f74bc7cd53080305f05358936f3a1b6dd4d91203f4cc799e81761cf5c53c5bbe9dcc13bdb27ec8f57ecf21b2f9ceec3c8d27"
ARG GRADLE_DOWNLOADS_SHA256="312eb12875e1747e05c2f81a4789902d7e4ec5defbd1eefeaccc08acf096505d 7.6"
ARG SBT_DOWNLOAD_SHA256="1f65344da074dbd66dfefa93c0eff8d319d772e5cad47fcbeb6ae178bbdf4686"
ENV LOG4J_UNSAFE_VERSIONS="2.11.1 1.2.8"

ARG MAVEN_CONFIG_HOME="/root/.m2"

ENV JAVA_HOME="$JAVA_17_HOME" \
    JDK_HOME="$JDK_17_HOME" \
    JRE_HOME="$JRE_17_HOME"

ENV PATH="${PATH}:/opt/tools"

RUN set -ex \
    && apt-get update \
    && apt-get install -y -qq software-properties-common apt-utils \
    # Install Corretto 17
    && wget -qO- https://apt.corretto.aws/corretto.key | apt-key add - \
    && add-apt-repository 'deb https://apt.corretto.aws stable main' \
    && apt-get update \
    && apt-get install -y -qq java-17-amazon-corretto-jdk \
    && apt-get install -y -qq --no-install-recommends ca-certificates-java \
    # Ensure Java cacerts symlink points to valid location
    && update-ca-certificates -f \
    && dpkg --add-architecture i386 \
    && apt-get update \
    && for tool_path in $JAVA_HOME/bin/*; do \
          tool=`basename $tool_path`; \
          update-alternatives --install /usr/bin/$tool $tool $tool_path 10000; \
          update-alternatives --set $tool $tool_path; \
        done \
     && rm $JAVA_HOME/lib/security/cacerts && ln -s /etc/ssl/certs/java/cacerts $JAVA_HOME/lib/security/cacerts \
    # Install Ant
    && curl -LSso /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz https://archive.apache.org/dist/ant/binaries/apache-ant-$ANT_VERSION-bin.tar.gz  \
    && echo "$ANT_DOWNLOAD_SHA512 /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar -xzf /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz -C /opt \
    && rm /var/tmp/apache-ant-$ANT_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/ant ant /opt/apache-ant-$ANT_VERSION/bin/ant 10000

RUN set -ex \
    # Install Maven
    && mkdir -p $MAVEN_HOME \
    && curl -LSso /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz https://archive.apache.org/dist/maven/maven-3/$MAVEN_VERSION/binaries/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && echo "$MAVEN_DOWNLOAD_SHA512 /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz" | sha512sum -c - \
    && tar xzf /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz -C $MAVEN_HOME --strip-components=1 \
    && rm /var/tmp/apache-maven-$MAVEN_VERSION-bin.tar.gz \
    && update-alternatives --install /usr/bin/mvn mvn /opt/maven/bin/mvn 10000 \
    && mkdir -p $MAVEN_CONFIG_HOME \
    # Install Gradle
    && mkdir -p $GRADLE_PATH \
    && for version in $INSTALLED_GRADLE_VERSIONS; do { \
       wget -q "https://services.gradle.org/distributions/gradle-$version-all.zip" -O "$GRADLE_PATH/gradle-$version-all.zip" \
       && unzip -q "$GRADLE_PATH/gradle-$version-all.zip" -d /usr/local \
       && echo "$GRADLE_DOWNLOADS_SHA256" | grep "$version" | sed "s|$version|$GRADLE_PATH/gradle-$version-all.zip|" | sha256sum -c - \
       && rm "$GRADLE_PATH/gradle-$version-all.zip" \
       && mkdir "/tmp/gradle-$version" \
       && "/usr/local/gradle-$version/bin/gradle" -p "/tmp/gradle-$version" init \
       && "/usr/local/gradle-$version/bin/gradle" -p "/tmp/gradle-$version" wrapper \
       # Android Studio uses the "-all" distribution for it's wrapper script.
       && perl -pi -e "s/gradle-$version-bin.zip/gradle-$version-all.zip/" "/tmp/gradle-$version/gradle/wrapper/gradle-wrapper.properties" \
       && "/tmp/gradle-$version/gradlew" -p "/tmp/gradle-$version" init \
       && rm -rf "/tmp/gradle-$version" \
       && if [ "$version" != "$GRADLE_VERSION" ]; then rm -rf "/usr/local/gradle-$version"; fi; \
     }; done \
    # Install default GRADLE_VERSION to path
    && ln -s /usr/local/gradle-$GRADLE_VERSION/bin/gradle /usr/bin/gradle \
    && rm -rf $GRADLE_PATH \
    # Install SBT
    && curl -fSL "https://github.com/sbt/sbt/releases/download/v${SBT_VERSION}/sbt-${SBT_VERSION}.tgz" -o sbt.tgz \
    && echo "${SBT_DOWNLOAD_SHA256} *sbt.tgz" | sha256sum -c - \
    && tar xzf sbt.tgz -C /usr/local/bin/ \
    && rm sbt.tgz \
    && for version in $LOG4J_UNSAFE_VERSIONS; do find / -name log4j*-$version.jar | xargs rm -f; done

ENV PATH "/usr/local/bin/sbt/bin:$PATH"
RUN sbt version -Dsbt.rootdir=true
# Cleanup
RUN rm -fr /var/lib/apt/lists/* /tmp/* /var/tmp/* \
    && apt-get clean
#****************     END JAVA     ****************************************************

#****************        DOCKER    *********************************************
ARG DOCKER_BUCKET="download.docker.com"
ARG DOCKER_CHANNEL="stable"
ARG DIND_COMMIT="3b5fac462d21ca164b3778647420016315289034"
ARG DOCKER_COMPOSE_VERSION="2.6.1"
ARG SRC_DIR="/usr/src"

ARG DOCKER_SHA256="945C3A3DDCB79EE7307496C2F39EB3D8372466E8654E63D60BBB462E4A3C1427"
ARG DOCKER_VERSION="20.10.22"

# Install Docker
RUN set -ex \
    && curl -fSL "https://${DOCKER_BUCKET}/linux/static/${DOCKER_CHANNEL}/x86_64/docker-${DOCKER_VERSION}.tgz" -o docker.tgz \
    && echo "${DOCKER_SHA256} *docker.tgz" | sha256sum -c - \
    && tar --extract --file docker.tgz --strip-components 1  --directory /usr/local/bin/ \
    && rm docker.tgz \
    && docker -v \
    # set up subuid/subgid so that "--userns-remap=default" works out-of-the-box
    && addgroup dockremap \
    && useradd -g dockremap dockremap \
    && echo 'dockremap:165536:65536' >> /etc/subuid \
    && echo 'dockremap:165536:65536' >> /etc/subgid \
    && wget -q "https://raw.githubusercontent.com/docker/docker/${DIND_COMMIT}/hack/dind" -O /usr/local/bin/dind \
    && curl -L https://github.com/docker/compose/releases/download/v${DOCKER_COMPOSE_VERSION}/docker-compose-Linux-x86_64 > /usr/local/bin/docker-compose \
    && chmod +x /usr/local/bin/dind /usr/local/bin/docker-compose \
    # Ensure docker-compose works
    && docker-compose version

VOLUME /var/lib/docker
#*********************** END  DOCKER  ****************************

#=======================End of layer: corretto  =================
FROM runtimes_n_corretto AS std_v6

# Activate runtime versions specific to image version.
RUN n $NODE_16_VERSION
RUN pyenv  global $PYTHON_310_VERSION
RUN phpenv global $PHP_81_VERSION
RUN rbenv  global $RUBY_31_VERSION
RUN goenv global  $GOLANG_18_VERSION

# Configure SSH
COPY ssh_config /root/.ssh/config
COPY runtimes.yml /codebuild/image/config/runtimes.yml
COPY dockerd-entrypoint.sh /usr/local/bin/dockerd-entrypoint.sh
COPY legal/bill_of_material.txt /usr/share/doc/bill_of_material.txt
COPY amazon-ssm-agent.json /etc/amazon/ssm/amazon-ssm-agent.json

ENTRYPOINT ["/usr/local/bin/dockerd-entrypoint.sh"]

#=====================Install chrome driver===========
FROM std_v6 AS chrome_driver
RUN echo "Installing Chrome"
      # # https://github.com/aws/aws-codebuild-docker-images/blob/48b53e25cf36dcb1e0b124bc7bebfb342d161b91/ubuntu/standard/5.0/Dockerfile#L74
RUN wget -q -O - https://dl.google.com/linux/linux_signing_key.pub | apt-key add -
RUN echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" | tee /etc/apt/sources.list.d/google.list
RUN apt-get -qq update
RUN apt-get -qq install -y google-chrome-stable
RUN google-chrome --version
      # # https://github.com/aws/aws-codebuild-docker-images/blob/48b53e25cf36dcb1e0b124bc7bebfb342d161b91/ubuntu/standard/5.0/Dockerfile#L82
RUN CHROME_VERSION=`google-chrome --version | awk -F '[ .]' '{print $3"."$4"."$5}'`
RUN CHROME_DRIVER_VERSION=`wget -qO- chromedriver.storage.googleapis.com/LATEST_RELEASE_114.0.5735`
RUN wget -qO /tmp/chromedriver_linux64.zip https://chromedriver.storage.googleapis.com/114.0.5735.16/chromedriver_linux64.zip
RUN unzip -q /tmp/chromedriver_linux64.zip -d /opt
RUN rm /tmp/chromedriver_linux64.zip
RUN mv /opt/chromedriver /opt/chromedriver-114.0.5735.16
RUN chmod 755 /opt/chromedriver-114.0.5735.16
RUN ln -s /opt/chromedriver-114.0.5735.16 /usr/bin/chromedriver
RUN chromedriver --version

#=====================Cache NPM packages===============
FROM chrome_driver AS npm_cache

RUN npm install lodash@latest chalk@latest request@latest commander@latest react@latest express@latest debug@latest async@latest fs-extra@latest moment@latest prop-types@latest react-dom@latest bluebird@latest underscore@latest vue@latest axios@latest tslib@latest mkdirp@latest glob@latest yargs@latest colors@latest inquirer@latest webpack@latest uuid@latest classnames@latest minimist@latest body-parser@latest rxjs@latest babel-runtime@latest jquery@latest yeoman-generator@latest through2@latest babel-core@latest core-js@latest semver@latest babel-loader@latest cheerio@latest rimraf@latest q@latest eslint@latest css-loader@latest shelljs@latest dotenv@latest typescript@latest @types/node@latest @angular/core@latest js-yaml@latest style-loader@latest winston@latest @angular/common@latest redux@latest object-assign@latest zone.js@latest babel-eslint@latest gulp@latest gulp-util@latest file-loader@latest ora@latest node-fetch@latest @angular/platform-browser@latest @babel/runtime@latest handlebars@latest eslint-plugin-import@latest @angular/compiler@latest eslint-plugin-react@latest aws-sdk@latest yosay@latest url-loader@latest @angular/forms@latest webpack-dev-server@latest @angular/platform-browser-dynamic@latest mocha@latest html-webpack-plugin@latest socket.io@latest ws@latest babel-preset-es2015@latest postcss-loader@latest node-sass@latest ember-cli-babel@latest babel-polyfill@latest @angular/router@latest ramda@latest react-redux@latest @babel/core@latest @angular/http@latest ejs@latest coffee-script@latest superagent@latest request-promise@latest autoprefixer@latest path@latest mongodb@latest chai@latest mongoose@latest xml2js@latest bootstrap@latest jest@latest sass-loader@latest redis@latest vue-router@latest chokidar@latest co@latest eslint-plugin-jsx-a11y@latest nan@latest optimist@latest promise@latest @angular/animations@latest postcss@latest morgan@latest less@latest immutable@latest qs@latest loader-utils@latest fs@latest extract-text-webpack-plugin@latest marked@latest mime@latest @alifd/next@latest meow@latest styled-components@latest resolve@latest reflect-metadata@latest babel-preset-react@latest jsonwebtoken@latest react-router-dom@latest extend@latest cookie-parser@latest whatwg-fetch@latest babel-preset-env@latest babel-jest@latest mysql@latest joi@latest minimatch@latest eslint-loader@latest react-dev-utils@latest node-uuid@latest es6-promise@latest cross-spawn@latest case-sensitive-paths-webpack-plugin@latest uglify-js@latest cors@latest eslint-plugin-flowtype@latest react-router@latest @babel/preset-env@latest deepmerge@latest socket.io-client@latest npm@latest webpack-manifest-plugin@latest koa@latest isomorphic-fetch@latest babel-cli@latest del@latest postcss-flexbugs-fixes@latest compression@latest update-notifier@latest babel-preset-react-app@latest jade@latest prompt@latest gulp-rename@latest angular@latest underscore.string@latest graphql@latest execa@latest browserify@latest opn@latest validator@latest eslint-config-react-app@latest vuex@latest prettier@latest invariant@latest jsdom@latest @types/react@latest redux-thunk@latest mini-css-extract-plugin@latest globby@latest pg@latest got@latest ajv@latest xtend@latest ember-cli-htmlbars@latest babel-plugin-transform-runtime@latest nodemailer@latest source-map-support@latest express-session@latest d3@latest less-loader@latest fsevents@latest babel-preset-stage-0@latest download-git-repo@latest query-string@latest font-awesome@latest open@latest passport@latest @types/lodash@latest grunt@latest path-to-regexp@latest mustache@latest inherits@latest tmp@latest md5@latest dotenv-expand@latest crypto-js@latest request-promise-native@latest through@latest connect@latest raf@latest react-scripts@latest readable-stream@latest highlight.js@latest @babel/polyfill@latest progress@latest optimize-css-assets-webpack-plugin@latest iconv-lite@latest bunyan@latest gulp-uglify@latest koa-router@latest ncp@latest lodash.merge@latest lru-cache@latest moment-timezone@latest figlet@latest history@latest readline-sync@latest pluralize@latest url@latest log4js@latest cli-table@latest webpack-merge@latest archiver@latest babel-register@latest eslint-config-airbnb@latest clone@latest jsonfile@latest puppeteer@latest shortid@latest @babel/plugin-proposal-class-properties@latest querystring@latest serve-static@latest tslint@latest pug@latest config@latest source-map@latest antd@latest concat-stream@latest element-ui@latest lodash.get@latest @babel/preset-react@latest serve-favicon@latest stylus@latest date-fns@latest esprima@latest sequelize@latest babel-plugin-transform-object-rest-spread@latest bindings@latest events@latest graceful-fs@latest normalize.css@latest crypto@latest cross-env@latest mime-types@latest event-stream@latest hoist-non-react-statics@latest gulp-concat@latest terser-webpack-plugin@latest json-loader@latest warning@latest bignumber.js@latest eventemitter3@latest webpack-cli@latest strip-ansi@latest cli-color@latest form-data@latest web3@latest gulp-sourcemaps@latest webpack-dev-middleware@latest ip@latest camelcase@latest sw-precache-webpack-plugin@latest merge@latest http-proxy@latest react-transition-group@latest multer@latest deep-equal@latest browser-sync@latest babel@latest dateformat@latest postcss-preset-env@latest uglifyjs-webpack-plugin@latest @polymer/polymer@latest sinon@latest eslint-config-prettier@latest gulp-sass@latest identity-obj-proxy@latest ts-loader@latest react-hot-loader@latest sqlite3@latest popper.js@latest which@latest markdown-it@latest tar@latest vue-template-compiler@latest babel-plugin-transform-class-properties@latest js-beautify@latest log-symbols@latest webpack-hot-middleware@latest rollup@latest copy-webpack-plugin@latest nodemon@latest boom@latest xmldom@latest recompose@latest util@latest ini@latest pify@latest command-line-args@latest vinyl@latest mz@latest lodash.debounce@latest html-minifier@latest ts-node@latest nconf@latest recursive-readdir@latest vue-loader@latest @types/express@latest datafire@latest @types/react-dom@latest babel-plugin-transform-decorators-legacy@latest clean-css@latest hoek@latest cookie@latest @babel/plugin-transform-runtime@latest when@latest babel-plugin-named-asset-import@latest postcss-safe-parser@latest bcrypt@latest @material-ui/core@latest @babel/plugin-syntax-dynamic-import@latest nunjucks@latest eslint-plugin-promise@latest react-native@latest lodash.isequal@latest workbox-webpack-plugin@latest acorn@latest amqplib@latest @svgr/webpack@latest color@latest ms@latest js-cookie@latest temp@latest simple-git@latest cssnano@latest reselect@latest yamljs@latest ioredis@latest koa-static@latest react-app-polyfill@latest react-select@latest escape-string-regexp@latest firebase@latest bn.js@latest escodegen@latest react-bootstrap@latest babelify@latest helmet@latest nopt@latest eslint-plugin-prettier@latest jest-resolve@latest knex@latest pnp-webpack-plugin@latest gulp-if@latest assert@latest global@latest npmlog@latest backbone@latest graphql-tag@latest raw-loader@latest run-sequence@latest lodash.clonedeep@latest @oclif/command@latest http-proxy-middleware@latest gulp-babel@latest @oclif/config@latest vinyl-fs@latest lodash.throttle@latest passport-local@latest eventemitter2@latest mqtt@latest unique-random-array@latest buffer@latest redux-saga@latest react-router-redux@latest jszip@latest koa-bodyparser@latest async-validator@latest babel-preset-stage-2@latest node-notifier@latest eslint-config-airbnb-base@latest material-ui@latest validate-npm-package-name@latest clean-webpack-plugin@latest hammerjs@latest redux-logger@latest htmlparser2@latest html-loader@latest filesize@latest gulp-plumber@latest consolidate@latest pkginfo@latest serialport@latest clear@latest should@latest json5@latest change-case@latest @babel/plugin-proposal-object-rest-spread@latest eslint-plugin-node@latest app-root-path@latest create-react-class@latest postcss-import@latest @angular/cdk@latest webpack-bundle-analyzer@latest JSONStream@latest pump@latest babylon@latest mobx@latest adm-zip@latest deep-extend@latest rc@latest http@latest @angular/material@latest eslint-config-standard@latest eslint-plugin-standard@latest once@latest numeral@latest @typescript-eslint/parser@latest prismjs@latest hapi@latest url-parse@latest @babel/cli@latest eslint-plugin-react-hooks@latest plugin-error@latest @typescript-eslint/eslint-plugin@latest require-dir@latest gulp-autoprefixer@latest url-join@latest istanbul@latest echarts@latest bower@latest resize-observer-polyfill@latest bcryptjs@latest lodash.assign@latest gm@latest babel-plugin-add-module-exports@latest diff@latest argparse@latest react-helmet@latest stylelint@latest string@latest protobufjs@latest find-up@latest esm@latest sprintf-js@latest rollup-plugin-node-resolve@latest configstore@latest download@latest traverse@latest websocket@latest codemirror@latest bs58@latest googleapis@latest method-override@latest formidable@latest tape@latest karma@latest json-stringify-safe@latest elasticsearch@latest split@latest safe-buffer@latest rsvp@latest @oclif/plugin-help@latest github@latest xlsx@latest tinycolor2@latest lodash-es@latest @babel/plugin-proposal-decorators@latest connect-history-api-fallback@latest @material-ui/icons@latest jsonschema@latest portfinder@latest fbjs@latest boxen@latest css@latest elliptic@latest sharp@latest http-errors@latest co-prompt@latest walk@latest restify@latest three@latest metalsmith@latest json-stable-stringify@latest webpack-node-externals@latest requirejs@latest regenerator-runtime@latest xmlbuilder@latest unzip@latest path-exists@latest rollup-plugin-commonjs@latest throttle-debounce@latest user-home@latest cron@latest xmlhttprequest@latest faker@latest systemjs@latest inflection@latest canvas@latest get-stdin@latest babel-types@latest read-pkg-up@latest graphql-tools@latest @types/jest@latest escape-html@latest broccoli-merge-trees@latest ssh2@latest sax@latest child_process@latest leaflet@latest wrench@latest rx@latest ethereumjs-util@latest jimp@latest enzyme@latest prettyjson@latest image-size@latest bfj@latest micromatch@latest gulp-watch@latest redux-actions@latest lit-element@latest discord.js@latest gulp-less@latest jshint@latest vinyl-source-stream@latest send@latest mysql2@latest @types/jquery@latest user@latest slash@latest es6-shim@latest watch@latest debounce@latest d3-scale@latest babel-helper-vue-jsx-merge-props@latest file-saver@latest pull-stream@latest lodash.omit@latest koa-compose@latest chart.js@latest gulp-replace@latest shallowequal@latest cli-spinner@latest object-path@latest react-intl@latest mobx-react@latest webpack-sources@latest flat@latest methods@latest jasmine@latest jest-watch-typeahead@latest react-dnd@latest object-hash@latest eslint-plugin-babel@latest parse5@latest friendly-errors-webpack-plugin@latest gulp-template@latest broccoli-funnel@latest @emotion/core@latest lodash.pick@latest listr@latest text-table@latest babel-plugin-transform-es2015-modules-commonjs@latest lodash.set@latest watchify@latest estraverse@latest unist-util-visit@latest @fortawesome/fontawesome-svg-core@latest @angular/compiler-cli@latest strip-json-comments@latest randomstring@latest node-emoji@latest react-addons-css-transition-group@latest clui@latest babel-plugin-import@latest tslint-react@latest errorhandler@latest blessed@latest electron@latest eslint-plugin-jest@latest bytes@latest deasync@latest vue-hot-reload-api@latest swig@latest jest-pnp-resolver@latest node-schedule@latest rollup-plugin-babel@latest figures@latest oauth@latest nedb@latest detect-port@latest liftoff@latest @types/fs-extra@latest get-port@latest stylelint-config-standard@latest gulp-imagemin@latest husky@latest @babel/register@latest gulp-install@latest gzip-size@latest node-gyp@latest node-forge@latest grunt-contrib-clean@latest pm2@latest plist@latest polished@latest @fortawesome/free-solid-svg-icons@latest ts-jest@latest react-test-renderer@latest grunt-contrib-watch@latest loglevel@latest on-finished@latest fs-promise@latest levelup@latest nomnom@latest @angular-devkit/core@latest react-dnd-html5-backend@latest react-motion@latest command-line-usage@latest readline@latest grunt-contrib-uglify@latest csv-parse@latest js-base64@latest level@latest dayjs@latest parseurl@latest babel-preset-stage-1@latest read@latest lodash.defaults@latest requireindex@latest tough-cookie@latest cli@latest emotion@latest map-stream@latest fancy-log@latest gulp-load-plugins@latest svgo@latest cross-fetch@latest is-plain-object@latest source-map-loader@latest @octokit/rest@latest multimatch@latest pretty-bytes@latest urijs@latest i18next@latest @babel/types@latest html-entities@latest cosmiconfig@latest pino@latest verror@latest gulp-notify@latest merge-stream@latest npm-run-all@latest stylus-loader@latest @babel/preset-typescript@latest grpc@latest he@latest supports-color@latest vue-style-loader@latest create-hash@latest touch@latest csv@latest tslint-config-prettier@latest lodash.camelcase@latest log-update@latest lodash.isplainobject@latest apollo-client@latest gulp-conflict@latest @types/cordova@latest mathjs@latest inversify@latest react-dropzone@latest yargs-parser@latest babel-traverse@latest babel-plugin-syntax-dynamic-import@latest raw-body@latest @babel/runtime-corejs2@latest sync-request@latest @types/jasmine@latest valid-url@latest react-tap-event-plugin@latest babel-plugin-transform-react-remove-prop-types@latest draft-js@latest cuid@latest slug@latest bcrypt-nodejs@latest @babel/parser@latest jwt-decode@latest postcss-cssnext@latest react-icons@latest install@latest basic-auth@latest memory-fs@latest d3-selection@latest @angular-devkit/schematics@latest es6-promisify@latest react-modal@latest redux-form@latest lodash.uniq@latest gulp-postcss@latest karma-chrome-launcher@latest fast-glob@latest file-type@latest child-process-promise@latest babel-plugin-transform-react-jsx@latest phantomjs-prebuilt@latest @angular/platform-server@latest brfs@latest tar-fs@latest clipboard@latest nyc@latest phantomjs@latest vinyl-buffer@latest clipboardy@latest lodash.isfunction@latest btoa@latest rxjs-compat@latest fork-ts-checker-webpack-plugin@latest @types/request@latest recast@latest d3-array@latest gulp-jshint@latest base-64@latest make-dir@latest https@latest sanitize-filename@latest bip39@latest vue-class-component@latest needle@latest keycode@latest grunt-contrib-copy@latest selenium-webdriver@latest opener@latest common-tags@latest is-wsl@latest koa-body@latest lodash.isstring@latest lodash.template@latest nodegit@latest command-exists@latest fstream@latest node-cache@latest node-watch@latest convert-source-map@latest @types/uuid@latest imagemin@latest gulp-filter@latest vue-property-decorator@latest gaze@latest supertest@latest stack-trace@latest gulp-clean-css@latest chance@latest gulp-typescript@latest lowdb@latest generic-pool@latest assert-plus@latest eslint-plugin-vue@latest gulp-minify-css@latest is-url@latest urllib@latest babel-generator@latest immer@latest sha1@latest grunt-contrib-jshint@latest mockjs@latest node.extend@latest connect-redis@latest babel-plugin-module-resolver@latest prompts@latest memoize-one@latest shell-quote@latest hyperquest@latest https-proxy-agent@latest long@latest chai-as-promised@latest secp256k1@latest http-server@latest utf8@latest type-is@latest vorpal@latest rollup-pluginutils@latest finalhandler@latest tweetnacl@latest apollo-link@latest coveralls@latest d3-shape@latest postcss-normalize@latest showdown@latest promise-polyfill@latest node-pre-gyp@latest path-is-absolute@latest @fortawesome/react-fontawesome@latest preact@latest crc@latest babel-plugin-transform-async-to-generator@latest osenv@latest hiredis@latest lodash.isempty@latest @babel/traverse@latest serve-index@latest lodash.map@latest highland@latest localforage@latest winston-daily-rotate-file@latest deep-diff@latest passport-oauth@latest passport-strategy@latest iview@latest dom-helpers@latest tildify@latest ethereumjs-tx@latest babel-template@latest leveldown@latest gulp-eslint@latest rc-util@latest depd@latest content-type@latest nanoid@latest babel-plugin-syntax-jsx@latest bl@latest read-pkg@latest markdown@latest jest-environment-jsdom-fourteen@latest react-error-overlay@latest serialize-javascript@latest nprogress@latest accepts@latest bizcharts@latest require-all@latest base64-js@latest component-emitter@latest connect-flash@latest koa-logger@latest react-color@latest findup-sync@latest passport-oauth2@latest dockerode@latest enzyme-adapter-react-16@latest etag@latest immutability-helper@latest grunt-cli@latest front-matter@latest mssql@latest react-native-vector-icons@latest react-virtualized@latest fork-ts-checker-webpack-plugin-alt@latest slugify@latest node-dir@latest babel-plugin-transform-regenerator@latest keypress@latest @types/bluebird@latest lodash.foreach@latest grunt-contrib-concat@latest split2@latest extend-shallow@latest @phosphor/widgets@latest xregexp@latest awesome-typescript-loader@latest hash-sum@latest apollo-link-http@latest babel-preset-stage-3@latest restler@latest lodash.flatten@latest react-apollo@latest intl@latest babel-plugin-dynamic-import-node@latest uglify-es@latest react-lifecycles-compat@latest lint-staged@latest @babel/plugin-proposal-export-default-from@latest normalize-wheel@latest is-promise@latest pako@latest pngjs@latest utils-merge@latest gray-matter@latest postcss-nested@latest pretty-error@latest rc-slider@latest node-static@latest decompress@latest gh-pages@latest cli-table2@latest os@latest reactstrap@latest sanitize-html@latest extract-zip@latest gulp-shell@latest vue-i18n@latest @types/mocha@latest http-status-codes@latest hogan.js@latest quill@latest soap@latest randombytes@latest decamelize@latest bson@latest imagemin-pngquant@latest arrify@latest bytebuffer@latest xml2json@latest lodash.isobject@latest columnify@latest fluent-ffmpeg@latest xml-js@latest sockjs-client@latest @emotion/styled@latest resolve-from@latest es6-error@latest react-slick@latest typeorm@latest sprintf@latest apollo-cache-inmemory@latest babel-plugin-lodash@latest hexlet-pairs@latest pretty-ms@latest pouchdb@latest md5-file@latest react-markdown@latest is@latest yeoman-environment@latest progress-bar-webpack-plugin@latest left-pad@latest stylelint-order@latest follow-redirects@latest stylelint-scss@latest koa-mount@latest busboy@latest sinon-chai@latest jasmine-core@latest loose-envify@latest process@latest envify@latest bufferutil@latest normalize-url@latest angular-animate@latest tv4@latest karma-mocha@latest karma-phantomjs-launcher@latest ts-pnp@latest atob@latest connect-mongo@latest @jupyterlab/application@latest @babel/plugin-proposal-export-namespace-from@latest http-status@latest bulma@latest cli-ux@latest iniparser@latest nock@latest object.assign@latest flux@latest semantic-ui-react@latest fastclick@latest karma-jasmine@latest fuse.js@latest inert@latest jwt-simple@latest isobject@latest jsdoc@latest element-resize-detector@latest react-datepicker@latest standard@latest react-onclickoutside@latest react-addons-shallow-compare@latest dot@latest browserslist@latest firebase-admin@latest nib@latest fuzzy@latest word-wrap@latest ansi-colors@latest tracer@latest yaml@latest ansi-escapes@latest jquery-ui@latest raven@latest i@latest xhr@latest deep-assign@latest inquirer-autocomplete-prompt@latest xpath@latest fresh@latest
#=======================END of STD:6.0  =================
