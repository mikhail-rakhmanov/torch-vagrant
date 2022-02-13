$script_build = <<-SCRIPT
git config --global credential.helper store
cat <<EOF > $HOME/.git-credentials
https://$GITHUB_USER:$GITHUB_TOKEN@github.com
EOF
git clone --recursive https://github.com/mikhail-rakhmanov/torch-lua52.git $HOME/torch
git config --global --unset credential.helper
rm $HOME/.git-credentials
docker build $HOME/torch
SCRIPT

$script_run = <<-SCRIPT
systemctl enable docker.service
mkdir $HOME/.docker
cat <<EOF > $HOME/.docker/config.json
{
        "auths": {
                "https://index.docker.io/v1/": {
                        "auth": "$DOCKER_AUTH"
                }
        }
}
EOF
docker pull mlrakhmanov/torch:1.0
docker logout
docker run -dit --restart unless-stopped --name torch -v /vagrant:/data mlrakhmanov/torch:1.0
cat <<'REOF' >> /home/vagrant/.bashrc

if [[ -n $SSH_CONNECTION ]] && [[ $USER == vagrant ]]; then
  cat <<EOF | docker exec -i torch sh
  cat /root/torch/install/share/lua/5.2/torch/Tensor.lua | python3 -c \
'import sys, re; \
s = sys.stdin.read(); \
s = re.sub(r"(?<=^\\s{9}else\\s{13}format\\s=\\s\\"%SZ\\.)4", "32", s, 1, re.MULTILINE); \
s = re.sub(r"(?<=^\\s{3}local\\snColumnPerLine\\s=\\s).*", "self:size(2)", s, 1, re.MULTILINE); \
print(s);' | tee /root/torch/install/share/lua/5.2/torch/Tensor.lua 1> /dev/null
EOF

  docker exec -it torch bash
fi
REOF
SCRIPT

Vagrant.configure("2") do |config|
  def auth(name, url, usr, passwd)
    def test_auth(url, usr, passwd)
      uri = URI.parse(url)
      request = Net::HTTP::Get.new(uri)
      request.basic_auth(usr, passwd)

      req_options = {
        use_ssl: uri.scheme == "https",
      }

      response = Net::HTTP.start(uri.hostname, uri.port, req_options) do |http|
        http.request(request)
      end
      return response.code
    end

    if usr == ""
      puts "Please insert your " + name + " credentials."
    end

    loop do
      while usr == "" do
        print "Username: "
        usr = STDIN.gets.chomp
      end
      if usr != "" and passwd == ""
        print "Personal access token: "
        passwd = STDIN.noecho(&:gets).chomp
        print "\n"
      end
      if test_auth(url, usr, passwd) == "401"
        puts "Invalid credentials."
        usr = ""
        passwd = ""
      elsif test_auth(url, usr, passwd) == "200"
        puts "Authorized on " + name + " as: " + usr
        break
      else
        print test_auth(url, usr, passwd)
        exit
      end
    end
  end

  if ARGV.include? '--provision-with' and ARGV.include? 'build' and ARGV.length > 2
    require 'net/http'
    require 'uri'

    $service = "GitHub"
    $address = "https://api.github.com/user"
    $user = ""
    $token = ""

    auth($service, $address, $user, $token)

  elsif ARGV.include? 'provision' or ARGV.include? '--provision-with' and ARGV.include? 'run' and ARGV.length > 2
    require 'net/http'
    require 'uri'
    require 'base64'

    $service = "DockerHub"
    $address = "https://index.docker.io/v1"
    $user = ""
    $token = ""

    auth($service, $address, $user, $token)

    $base = Base64.strict_encode64($user + ":" + $token)
  end

  config.vm.box = "ubuntu/focal64"
  config.vm.box_version = "20210907.0.0"
  config.vm.box_check_update = false

  config.vm.network "forwarded_port", id: "ssh", guest: 22, host: 2222, host_ip: "127.0.0.1"

  config.vm.provider "virtualbox" do |vb|
    vb.gui = false
    vb.memory = "2048"
    vb.cpus = "2"
    vb.check_guest_additions = false
  end

  config.vm.provision "build", type: "docker", run: "never" do |d|
    d.post_install_provision "shell", env: {"GITHUB_USER" => $user, "GITHUB_TOKEN" => $token}, sensitive: "true", inline: $script_build
  end

  config.vm.provision "run", type: "docker", run: "never" do |d|
    d.post_install_provision "shell", env: {"DOCKER_AUTH" => $base}, sensitive: "true", inline: $script_run
  end
end
