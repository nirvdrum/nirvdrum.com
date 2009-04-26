desc "deploy site to nirvdrum.com"
task :deploy do
  require 'rubygems'
  require 'highline/import'
  require 'net/ssh'

  branch = "master"

  username = ask("Username:  ") { |q| q.echo = true }
  password = ask("Password:  ") { |q| q.echo = "*" }

  Net::SSH.start('nirvdrum.com', username, :port => 22, :password => password) do |ssh|
    commands = <<EOF
cd /var/www/nirvdrum.com/cached-copy
git checkout #{branch}
git pull origin #{branch}
git checkout -f
jekyll --no-auto
rm -rf ../_old
mv ../www ../_old
mv _site ../www
echo "Done deploying"
EOF
    commands = commands.gsub(/\n/, "; ")
    ssh.exec commands
  end
end
