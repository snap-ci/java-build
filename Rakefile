require 'rubygems'
require 'bundler/setup'

require 'rake/clean'

distro = nil
fpm_opts = ""

if File.exist?('/etc/system-release') && File.read('/etc/redhat-release') =~ /centos|redhat|fedora|amazon/i
  distro = 'rpm'
  fpm_opts << " --rpm-user root --rpm-group root "
elsif File.exist?('/etc/os-release') && File.read('/etc/os-release') =~ /ubuntu|debian/i
  distro = 'deb'
  fpm_opts << " --deb-user root --deb-group root "
end

unless distro
  $stderr.puts "Don't know what distro I'm running on -- not sure if I can build!"
end

CLEAN.include("downloads")
CLEAN.include("jailed-root")
CLEAN.include("log")
CLEAN.include("pkg")
CLEAN.include("tmp")

{
  '1.6' => {:url => 'http://download.oracle.com/otn-pub/java/jdk/6u45-b06/jdk-6u45-linux-x64.bin',    :checksum => '40c1a87563c5c6a90a0ed6994615befe'},
  '1.7' => {:url => 'http://download.oracle.com/otn-pub/java/jdk/7u55-b13/jdk-7u55-linux-x64.tar.gz', :checksum => '9e1fb7936f0e5aaa1e64d36ba640bc1f'},
  '1.8' => {:url => 'http://download.oracle.com/otn-pub/java/jdk/8u5-b13/jdk-8u5-linux-x64.tar.gz',   :checksum => 'adc3827532741873de9216a5aed883ed'},
}.each do |version, description|
  namespace version do
    prefix      = File.join("/opt/local/java", version)
    java_jailed_root = File.join("jailed-root", prefix)

    url         = description[:url]
    checksum    = description[:checksum]
    java_source = File.basename(url)

    task :init do
      mkdir_p "log"
      mkdir_p "pkg"
      mkdir_p "downloads"
      mkdir_p "jailed-root"
    end

    task :download do
      cd 'downloads' do
        sh("curl --fail --location --cookie 'oraclelicense=accept-securebackup-cookie' #{url} > #{java_source} 2>/dev/null")
        sh("echo '#{checksum}  #{java_source}' > #{java_source}.md5")
        sh("md5sum --check --status #{java_source}.md5")
      end
    end

    task :unpack do
      rm_rf "jailed-root"
      mkdir_p java_jailed_root
      if java_source =~ /\.tar\.gz/
        sh("tar -zxf downloads/#{java_source} -C #{java_jailed_root} --strip-components=1")
      elsif java_source =~ /\.bin/
        sh("mkdir tmp; cd tmp; bash ../downloads/#{java_source}")
        sh("mv tmp/jdk*/* #{java_jailed_root}")
      end
    end

    task :fpm do
      mkdir_p "pkg"
      description_string = %Q{The Java development tools.}
      release = Time.now.utc.strftime('%Y%m%d%H%M%S')
      cd "pkg" do
        sh(%Q{
          bundle exec fpm -s dir -t #{distro} --name sun-java-#{version} -a x86_64 --version "#{version}" -C ../jailed-root --directories #{prefix} --verbose #{fpm_opts} --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'Oracle Binary Code License Agreement' .
        })
      end
    end

    desc "build and package ruby-#{version}"
    task :all => [:clean, :init, :download, :unpack, :fpm]
  end

  task :default => "#{version}:all"
end

desc "build all rubies"
task :default
