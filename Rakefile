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
  '1.6' => {
    :url      => 'http://download.oracle.com/otn-pub/java/jdk/6u45-b06/jdk-6u45-linux-x64.bin',
    :checksum => '40c1a87563c5c6a90a0ed6994615befe',
    :exclude  => [
      './man/',         # man pages
      './db/',          # derby
      './src.zip',      # the jdk sources
      './lib/visualvm', # visualvm
    ]
  },
  '1.7' => {
    :url      => 'http://download.oracle.com/otn-pub/java/jdk/7u80-b15/jdk-7u80-linux-x64.tar.gz',
    :checksum => '6152f8a7561acf795ca4701daa10a965',
    :exclude  => [
      './man/',         # man pages
      './db/',          # derby
      './src.zip',      # the jdk sources
      './lib/visualvm', # visualvm
    ]
  },
  '1.8' => {
    :url      => 'http://download.oracle.com/otn-pub/java/jdk/8u51-b16/jdk-8u51-linux-x64.tar.gz',
    :checksum => 'b34ff02c5d98b6f372288c17e96c51cf',
    :exclude  => [
      './man/',               # man pages
      './db/',                # derby
      './src.zip',            # the jdk sources
      './lib/visualvm',       # visualvm
      './lib/missioncontrol', # missioncontrol
    ]
  }
}.each do |version, description|
  namespace version do
    prefix      = File.join("/opt/local/java", version)
    java_jailed_root = File.join("jailed-root", prefix)

    url         = description[:url]
    checksum    = description[:checksum]
    excludes    = description[:exclude]
    java_source = File.basename(url)

    task :init do
      mkdir_p 'log'
      mkdir_p 'pkg'
      mkdir_p 'downloads'
      mkdir_p 'jailed-root'
    end

    task :download do
      cd 'downloads' do
        sh("curl --fail --location --cookie 'oraclelicense=accept-securebackup-cookie' #{url} > #{java_source} 2>/dev/null")
        sh("echo '#{checksum}  #{java_source}' > #{java_source}.md5")
        sh("md5sum --check --status #{java_source}.md5")
      end
    end

    task :unpack do
      rm_rf 'jailed-root'
      mkdir_p java_jailed_root
      if java_source =~ /\.tar\.gz/
        sh("tar -zxf downloads/#{java_source} -C #{java_jailed_root} --strip-components=1")
      elsif java_source =~ /\.bin/
        sh("mkdir tmp; cd tmp; bash ../downloads/#{java_source}")
        sh("mv tmp/jdk*/* #{java_jailed_root}")
      end

      cd java_jailed_root do
        excludes.each do |exclude|
          rm_rf Dir[exclude]
        end
      end
    end

    task :fpm do
      mkdir_p 'pkg'
      description_string = %Q{The Java development tools.}
      release = Time.now.utc.strftime('%Y%m%d%H%M%S')
      cd 'pkg' do
        sh(%Q{
          bundle exec fpm -s dir -t #{distro} --name sun-java-#{version} -a x86_64 --version "#{version}" -C ../jailed-root --directories #{prefix} --verbose #{fpm_opts} --maintainer snap-ci@thoughtworks.com --vendor snap-ci@thoughtworks.com --url http://snap-ci.com --description "#{description_string}" --iteration #{release} --license 'Oracle Binary Code License Agreement' .
        })
      end
    end

    desc "build and package java-#{version}"
    task :all => [:clean, :init, :download, :unpack, :fpm]
  end

  task :default => "#{version}:all"
end

desc 'build all java versions'
task :default