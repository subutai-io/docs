
docs_port = 8000

Vagrant.configure("2") do |config|
  config.vm.box = "subutai/stretch"

  config.vm.network "forwarded_port", guest: 8000, host: 8000, auto_correct: true

  config.vm.provision 'shell',
    env: {
      "ACNG_HOST" => ENV['ACNG_HOST'],
      "ACNG_PORT" => ENV['ACNG_PORT'],
      "APPROX_HOST" => ENV['APPROX_HOST'],
      "APPROX_PORT" => ENV['APPROX_PORT'],
      "GDRIVE_TOKEN_FILE" => ENV['GDRIVE_TOKEN_FILE'],
      "GDRIVE_RTD_ROOT" => ENV['GDRIVE_RTD_ROOT']
      }, inline: <<-SHELL

    # Tuck these away into the environment so they're available to synchronize
    echo "GDRIVE_RTD_ROOT=$GDRIVE_RTD_ROOT" >> /etc/environment

    ## Need to make this conditional
    ACNG_URL="http://$ACNG_HOST:$ACNG_PORT"
    APPROX_URL="http://$APPROX_HOST:$APPROX_PORT/debian/"

    # Apt settings
    echo 'Using '$ACNG_URL' and '$APPROX_URL' for deb pkg caching'
    echo 'Acquire::http::Proxy "'$ACNG_URL'";' > /etc/apt/apt.conf.d/02proxy

    echo "Finding nearest apt mirror even when caching (not everything gets cached)"
    apt-get -y update
    apt-get install -y netselect-apt
    country=`curl -s ipinfo.io | grep country | awk -F ':' '{print $2}' | sed -e 's/[", ]//g'`
    if [ "KG" = "$country" ]; then
      country='KZ'
    fi

    netselect-apt -c $country &> /dev/null
    if [ ! "$?" = "0" ]; then
      netselect-apt -c US &> /dev/null
    fi

    if [ -f "sources.list" ]; then
      rm /etc/apt/sources.list

      while read line; do
        if [ -n "$(echo $line | egrep '^#.*')" -o -z "$(echo $line | grep '^deb .*')" ]; then
          continue;
        fi

        echo "$line non-free" >> /etc/apt/sources.list;
        echo "$line non-free" | sed -e 's/deb /deb-src /' >> /etc/apt/sources.list;
      done < sources.list
    fi

    apt-get -y update
    apt-get install -y git python python-pip python-dev build-essential
    pip install --upgrade pip
    pip install --upgrade virtualenv
    pip install sphinx sphinx-autobuild recommonmark
    pip install sphinx_rtd_theme

    # Get the google drive client
    wget 'https://docs.google.com/uc?id=0B3X9GlR6EmbnQ0FtZmJJUXEyRTA&export=download' -O /usr/local/bin/gdrive
    chmod +x /usr/local/bin/gdrive

    # Before exiting in privileged mode setup iptables and routing
    # sphinx-autobuild does not listen on all interface just localhost
    sysctl -w net.ipv4.conf.enp0s8.route_localnet=1
    iptables -t nat -I PREROUTING -p tcp -d 172.16.0.0/16 --dport 8000 -j DNAT --to-destination 127.0.0.1:8000
  SHELL
  
  if ENV['GDRIVE_TOKEN_FILE'].nil? && ARGV[1] == 'up'
    puts 'Cannot enable google drive client without a proper token file.'
    puts 'Set the path to the token file using the GDRIVE_TOKEN_FILE environment variable.'
    puts 'If you do not have a token file, then log into the VM and run \'gdrive about\''
    puts 'It will prompt you with a URL to go to and authenticate to get a verification code.'
    puts 'You provide this code, and it stores the bearer token in a token file.'
    puts 'Copy the bearer token file generated in the VM under /home/subutai/.gdrive with'
    puts 'the name \'token_v2.json\' to somewhere safe, like your hosts $HOME/.ssh and'
    puts 'set GDRIVE_TOKEN_FILE to its path. You can then reprovision using:'
    puts ''
    puts 'GDRIVE_TOKEN_FILE=$HOME/.ssh/token_v2.json vagrant provision'
    puts ''
    puts 'You can also export this environment variable to permanently set it in'
    puts 'your shell profile file. Then you do not have to provide this value on'
    puts 'every vagrant up commands.'
  else
    config.vm.provision 'file',
      source: "#{ENV['GDRIVE_TOKEN_FILE']}", 
      destination: '/tmp/token_v2.json'
  end

  config.vm.provision 'shell', privileged: false, 
  env: {
    "ACNG_HOST" => ENV['ACNG_HOST'],
    "ACNG_PORT" => ENV['ACNG_PORT'],
    "APPROX_HOST" => ENV['APPROX_HOST'],
    "APPROX_PORT" => ENV['APPROX_PORT'],
    "GDRIVE_TOKEN_FILE" => ENV['GDRIVE_TOKEN_FILE'],
    "GDRIVE_RTD_ROOT" => ENV['GDRIVE_RTD_ROOT']
    }, inline: <<-SHELL

    if [ -f /tmp/token_v2.json ]; then
      mkdir $HOME/.gdrive
      mv /tmp/token_v2.json $HOME/.gdrive
      gdrive about
    else
      echo 'Google drive client not initialized. Setup token file and reprovision.'
      echo 'Will not attempt site generation or start autobuild daemon.'
      echo 'Existing with non-zero status."
      exit 1
    fi

    cd /vagrant;
    nohup sphinx-autobuild . _build/html vagrant &
  SHELL
end
