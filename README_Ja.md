Wheneverは、クリアな構文を書いたり、デプロイするcronジョブを提供するジェムです。

インストール方法

$ gem install whenever
または、Gemfileに入れてBundlerでインストールしてください。

gem 'whenever', :require => false
入門

$ cd /apps/my-great-project
$ wheneverize .
これはあなたのための（config/schedule.rb）初期設定ファイルが作成されます。

schedule.rbファイルの例

every 3.hours do
  runner "MyModel.some_process"
  rake "my:rake:task"
  command "/usr/bin/my_great_command"
end

every 1.day, :at => '4:30 am' do
  runner "MyModel.task_to_run_at_four_thirty_in_the_morning"
end

every :hour do # Many shortcuts available: :hour, :day, :month, :year, :reboot
  runner "SomeModel.ladeeda"
end

every :sunday, :at => '12pm' do # Use any day of the week or :weekend, :weekday
  runner "Task.do_something_great"
end

every '0 0 27-31 * *' do
  command "echo 'you can use raw cron syntax too'"
end

# Capistranoで:appの役割のみを持つサーバーでタスクが実行されます。
# Capistranoの役割のセクションをみてください。
every :day, :at => '12:20am', :roles => [:app] do
  rake "app_server:task"
end
独自のジョブタイプを定義します。

Wheneverは、所定の3つのジョブタイプ（command、runner、rake）を渡せます。 
あなたはジョブタイプを使用して独自に定義ができます:

（例）
job_type :awesome, '/usr/local/bin/awesome :task :fun_level'

every 2.hours do
  awesome "party", :fun_level => "extreme"
end

 /usr/local/bin/awesome を2時間おきに実行します。
 :task は常にはじめの引数と入れ替わります,
 またいくつか追加の :whatevers とセットで定義されたオプションを入れ替わります。
 
 
付属する標準のジョブは、Wheneverのように定義されています。

job_type :command, ":task :output"
job_type :rake,    "cd :path && :environment_variable=:environment bundle exec rake :task --silent :output"
job_type :runner,  "cd :path && bin/rails runner -e :environment ':task' :output"
job_type :script,  "cd :path && :environment_variable=:environment bundle exec script/:task :output"
Pre-Railsの3つのアプリとアプリを正しく機能するには、Bundlerを再定義してrakeとrunner jobsをそれぞれで使わないでください、

もし:pathがセットされずに、デフォルトのディレクトリを実行した場合は、
:environment_variable デフォルトでproductionが設定されます。
また:outputの出力リダイレクトの置き換えについては、更にここの詳細を読むことができます。
here: http://github.com/javan/whenever/wiki/Output-redirection-aka-logging-your-cron-jobs

すべてのジョブはデフォルトで
bash -l -c 'command...'.で走査します。
とりわけ、このcronジョブはいい働きができ、RVMのcronは限定された環境の代わりに
環境内の全体を読み込むことができます。

詳細はこちら: http://blog.scoutapp.com/articles/2010/09/07/rvm-and-cron-in-production

あなた自身の:job_templateの設定を変更できます。
set :job_template, "bash -l -c ':job'"
もしくはjob_templateをnilからあなたのジョブを正常に実行できます。

set :job_template, nil
Capistrano 統合

Capistranoのbuilt-inは、更新やデプロイ、crontabも簡単に扱えます。
CapistranoのV3を対象に、次の項目を見ていきましょう。

では、あなたの"config/deploy.rb"ファイル：
require "whenever/capistrano"
設定できるオプションをレシピから見てみましょう。
https://github.com/javan/whenever/blob/master/lib/whenever/capistrano/v2/recipes.rb 参考例として、もしbundlerを使用しているならこちら

set :whenever_command, "bundle exec whenever"
require "whenever/capistrano"
もし異なる環境(staging、productionといった)を使っているのなら、あなたはこれを実行できます。

set :whenever_environment, defer { stage }
require "whenever/capistrano"
Capistranoの貴重な:stageは、お使いの環境名を保持していなければなりません。この:environmentで利用可能なschedule.rbを正しく実行できます。

もしどちらの環境も同じサーバーであれば、namespaceか上書きすることでdeployできます。


set :whenever_environment, defer { stage }
set :whenever_identifier, defer { "#{application}_#{stage}" }
require "whenever/capistrano"
Capistrano V3 統合

"Capfile" ファイルについて:

require "whenever/capistrano"

load:defaults (ファイル内の底)に設定できるオプションタスクがあるので見てみましょう。  https://github.com/javan/whenever/blob/master/lib/whenever/capistrano/v3/tasks/whenever.rake. 参考例、namespaceから
For example, to namespace the crontab entries by application and stage do this.

In your in "config/deploy.rb" file:

set :whenever_identifier, ->{ "#{fetch(:application)}_#{fetch(:stage)}" }
Capistrano roles

The first thing to know about the new roles support is that it is entirely optional and backwards-compatible. If you don't need different jobs running on different servers in your capistrano deployment, then you can safely stop reading now and everything should just work the same way it always has.

When you define a job in your schedule.rb file, by default it will be deployed to all servers in the whenever_roles list (which defaults to [:db]).

However, if you want to restrict certain jobs to only run on subset of servers, you can add a :roles => [...] argument to their definitions. Make sure to add that role to the whenever_roles list in your deploy.rb.

When you run cap deploy, jobs with a :roles list specified will only be added to the crontabs on servers with one or more of the roles in that list.

Jobs with no :roles argument will be deployed to all servers in the whenever_roles list. This is to maintain backward compatibility with previous releases of whenever.

So, for example, with the default whenever_roles of [:db], a job like this would be deployed to all servers with the :db role:

every :day, :at => '12:20am' do
  rake 'foo:bar'
end
If we set whenever_roles to [:db, :app] in deploy.rb, and have the following jobs in schedule.rb:

every :day, :at => '1:37pm', :roles => [:app] do
  rake 'app:task' # will only be added to crontabs of :app servers
end

every :hour, :roles => [:db] do
  rake 'db:task' # will only be added to crontabs of :db servers
end

every :day, :at => '12:02am' do
  command "run_this_everywhere" # will be deployed to :db and :app servers
end
Here are the basic rules:

If a server's role isn't listed in whenever_roles, it will never have jobs added to its crontab.
If a server's role is listed in the whenever_roles, then it will have all jobs added to its crontab that either list that role in their :roles arg or that don't have a :roles arg.
If a job has a :roles arg but that role isn't in the whenever_roles list, that job will not be deployed to any server.
RVM Integration

If your production environment uses RVM (Ruby Version Manager) you will run into a gotcha that causes your cron jobs to hang. This is not directly related to Whenever, and can be tricky to debug. Your .rvmrc files must be trusted or else the cron jobs will hang waiting for the file to be trusted. A solution is to disable the prompt by adding this line to your user rvm file in ~/.rvmrc

rvm_trust_rvmrcs_flag=1

This tells rvm to trust all rvmrc files.

The whenever command

$ cd /apps/my-great-project
$ whenever
This will simply show you your schedule.rb file converted to cron syntax. It does not read or write your crontab file. Run whenever --help for a complete list of options.

Credit

Whenever was created for use at Inkling (http://inklingmarkets.com). Their take on it: http://blog.inklingmarkets.com/2009/02/whenever-easy-way-to-do-cron-jobs-from.html

Thanks to all the contributors who have made it even better: http://github.com/javan/whenever/contributors

Discussion / Feedback / Issues / Bugs

For general discussion and questions, please use the google group: http://groups.google.com/group/whenever-gem

If you've found a genuine bug or issue, please use the Issues section on github: http://github.com/javan/whenever/issues

Ryan Bates created a great Railscast about Whenever: http://railscasts.com/episodes/164-cron-in-ruby It's a little bit dated now, but remains a good introduction.

Compatible with Ruby 1.8.7-2.2.0, JRuby, and Rubinius. Build Status

Copyright © 2015 Javan Makhmali
