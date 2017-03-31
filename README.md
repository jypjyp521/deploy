
基于gitolite自动化版本控制


环境（linux centos 65 x64 两台（192.168.1.112,192.168.1.119），windows一台）
一、192.168.1.119局域网安装gitolite
1、安装git
 Yum install git  -y
2、添加git用户
 Adduser git

3、生成git ssh公约私钥
  Su git
  Ssh-keygen
将公约推到112服务器上待用
 Scp /home/git/.ssh/id_rsa.pub root@192.168.1.112:/home/git/119.pub

4、安装gitolite
cd
mkdir bin
git clone https://github.com/sitaramc/gitolite.git 
./gitolite/install --to /home/git/bin/
bin/gitolite setup -pk local.pub 
/*****************local.pub 可由本地机器在windows下git客户端git bash 生成命令为ssh-keygen，放在C:\Users\Administrator\.ssh\id_rsa.pub,将其推到119机器上/home/git/local.pub**************************


5、在本地机器Clone管理版本库，并创建新的版本库
Git clone git@192.168.1.119:gitolite-admin
Cd gitolite-admin
Vim conf/gitolite.conf
Repo repo1
RW+		=	local

二、在外网测试服务器添加gitolite（192.168.1.112）	
1、安装git
 Yum install git  -y
2、添加git用户
  Adduser git
3、安装gitolite
Su git
Cd 
mkdir bin
git clone https://github.com/sitaramc/gitolite.git 
./gitolite/install --to /home/git/bin/

bin/gitolite setup -pk 119.pub 

三、在局域网服务器（192.168.1.119）
1、Clone管理版本库，并创建新的版本库
Git clone git@192.168.1.119:gitolite-admin
Cd gitolite-admin
Vim conf/gitolite.conf
Repo repo1
RW+		=	119
2、配置ssh验证

Vim /home/git/.ssh/config
Su root 
Chmod /home/git/.ssh/config 600

Su git
Cd 
Host	 git.server(可以自定义)
HostName	192.168.1.112
Port	22
User	git
IdentityFile	~/.ssh/id_rsa
2、添加远程版本库地址
Cd /home/git/repositories/repo1.git 
git remote set-url git.server git@git.server:repo1

3、添加git钩子同步版本到112外网服务器
Cd /home/git/repositories/repo1.git/hooks
cp post-receive.sample post-receive
Vim post-receive
while read oldrev newrev ref
do
        echo "STATRT $oldrev $newrev $ref";
        branch=${ref##refs/heads/};
        git push git.server $branch;
done;



四、在外网服务器（192.168.1.112）部署
1、添加发布钩子
Cd /home/git/repositories/repo1.git/hooks
cp post-receive.sample post-receive
vim post-receive

while read oldrev newrev ref
do
       echo "START $oldrev $newrev $ref";
       #if [[ $ref =~ "refs/tags/" ]]; then
		git --git-dir=/home/git/repositories/repo1.git diff ${oldrev}...${newrev} > /tmp/diff.txt;
		branch=${ref##refs/heads/};
		if [[ -n "$branch"  ]]; then
                         echo "$branch +++"
	
			/repo1/dev.config/deploy /repo1/dev/ ${branch} > /repo1/dev/data/deploy.log;
		else
                        echo "$branch ---"
			/repo1/dev.config/deploy /repo1/dev/ $newrev > /repo1/dev/data/deploy.log;
		fi
		#echo 'deploy dev done';

       #fi
done;
2、创建项目
Mkdir /repo1
Mkdir /repo1/dev
Mkdir /repo1/dev.config

Cd /repo1/dev.config
Vim deploy
#!/bin/sh

usage="usage:deploy src_path branch update_node";

src_path=$1;
selected_branch=$2;
update_node=$3;
current_branch="";
git_dir="/home/git/repositories/repo1.git";
git_work_tree=$src_path;
diff_file="/tmp/diff.txt";
deploy_file="$src_pathdata/deploy.log";
deploy_lock="$src_pathdata/deploy.lock";
config_dir="/repo1/dev.config/*";


if [[ "$src_path" == "-h" ]]; then
	echo $usage;
	exit;
fi

if [[ $# < 2 ]]; then
	echo $usage;
	exit;
fi

if [[ ! -d $src_path ]]; then
	echo $src_path;
	echo "source path is not exists.keep the path is absolute path";
	exit;
fi

function deploy()
	
	# 发送diff邮件
	# 记录当前的分支
	# current_hash;
	if [[ -f $deploy_lock ]]; then
		locked_branch=`cat $deploy_lock | cut -d \| -f 1`;
		locked=`cat $deploy_lock | cut -d \| -f 2`;
		if [[ $locked -gt 0 && "$locked_branch" != "$selected_branch" ]]; then
			echo 'auto deploy has LOCKED,if you WANT TO DEPLOY,visit /deploy.php to deploy';
			return 2;
		fi
	fi
	# 旧版本hash值
	echo "current deploy branch：$selected_branch";

	# 先初始化部分依赖 1.nodejs module
	if [[ -n $update_node ]]; then
		init;
	fi
	
	git --git-dir=$git_dir --work-tree=$git_work_tree checkout -f $selected_branch && git --git-dir=$git_dir --work-tree=$git_work_tree reset --hard;

	chmod -R 755 $src_path;
	
	# 生成配置文件
	generate_config;
	# copy 配置文件及其他文件
	echo "copy config files";
	#node $src_pathnode_modules/.bin/gulp --cwd $src_path init;
	/bin/cp -rf $config_dir $src_path;
	# 执行migrate数据库自动化脚本
	migrate;
	# 执行gulp自动化脚本
	#echo "GULP START";
	#node $src_pathnode_modules/.bin/gulp --cwd $src_path dev;
	#echo "GULP END";
	# 生成配置文件
	generate_config;

	return 0;


function diff_email()
        if [[ -f $send_file ]]; then
                /usr/bin/python /home/git/python/send_email.py $send_file;
                echo "$diff_file";
                #rm -f $diff_file;
        fi


function deploy_email()
	if [[ -f $deploy_file ]]; then
		node $src_pathnode_modules/.bin/gulp --cwd $src_path deploy_email;
	fi


function current_hash()
	HEAD=$(cat $src_path.git/HEAD);
	if [[ $HEAD =~ "ref:" ]]; then
		ref=$HEAD##ref: ;
		current_branch=$(cat $src_path.git/$ref);
	else
		current_branch=$HEAD;
	fi


function uniqid()
	echo `/alidata/server/php/bin/php -r "echo uniqid();"`;


function init()
	npm install;
	npm update;


function rollback()
	git checkout -f $current_hash;
	echo 'rollback success';


function migrate()
	/alidata/server/php/bin/php -c /alidata/server/php-5.4.23/etc/php.ini -f $src_pathapplication/migrate.php;


function gulp_script()
	echo 'gulp start';
	
	echo 'gulp end';


function generate_config()
	config_file=$src_pathapplication/config/env.php;
	configs=("\$config['version']='`uniqid`';");
	echo "<?php" > $config_file;
	for config in $configs; do
		echo "" >> $config_file;
		echo $config >> $config_file;
	done
	echo 'generate version config done';


deploy;

if [[ $? == 0 ]]; then
	echo 'deploy done';
elif [[ $? == 2 ]]; then
	echo 'push doen';
else
	# 如果出错,则回滚
	if [[ -f $backup_file ]]; then
		rollback;
	fi
	echo 'deploy failed';
fi


Vim deploy.php

<?php

header('Content-type:text/html;charset=utf-8');

error_reporting(E_ALL & ~E_NOTICE);
ini_set('display_error', 'Off');
// error_reporting(E_ALL);

date_default_timezone_set('Asia/Shanghai');

$src_path="/home/git/repositories/cunguan.git/";
$dest_path="/repo1/dev/";
$deploy_output_log="$dest_pathdata/deploy.log";
$deploy_lock_file="$dest_pathdata/deploy.lock";

if ($_SERVER['REQUEST_METHOD']=='POST') 
	$selected_branch=trim($_POST['branch']);
	if ($selected_branch) 
		exec($dest_path.“/dev/deploy $dest_path $selected_branch > $deploy_output_log &",$output);
        @file_put_contents($deploy_lock_file,sprintf('%s|%s',$selected_branch,(int)$_POST['locked']));
		header("Location:/deploy.php");
	


exec("git --git-dir=$src_path branch",$branches,$status);

$branches=array_map(function($value)
    return trim($value,"* ");
, $branches);

list($deploy_branch,$locked)=@explode('|', @file_get_contents($deploy_lock_file));

$deploy_output=@file_get_contents($deploy_output_log);

?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>deploy</title>
    <style type="text/css">
    	body
    		width:960px;
    		margin:auto;
    		padding:0;
    		font-size:14px;
    	
    	.control-group
    		width:100%;
    		margin:20px 15px;
    	
    	label
    		font-size:16px;
    		display:inline-block;
    		height:22px;
    		line-height:22px;
    		margin-top:15px;
    		margin-left:20px;
    	
    	.branch-label

    	
    	select
    		width:220px;
    		line-height:22px;
    	
    	textarea
			width: 600px;
			height: 300px;
			font-size:14px;
			line-height:20px;
    	
    	.deploy-result
		    vertical-align: top;
		    position: relative;
		    top: -10px;
    	
    	.deploy-btn
    		display:inline-block;
    		height:22px;
    		line-height:22px;
    		font-size:16px;
    		width:60px;
    	
    	.deploy-btn:first-child
    		margin-left:60px;
    	
    	.deploy-btn:last-child
    		margin-left:10px;
    	
    </style>
</head>
<body>
    <form class="form-horizontal" method="POST">
	    <div class="control-group">
	        <label class="control-label branch-label" style="display: inline-block;width: 80px;">分支</label>
	        <select class="input-xlarge" name="branch" onchange="change_branch(this)">
	            <?php foreach ($branches as $k=>$branch)  ?>
					<?php if (trim($deploy_branch)==trim(str_replace('*','',$branch)))  ?>
						<option selected name="<?php echo $branch; ?>"><?php echo $branch; ?></option>
					<?php else ?>
						<option name="<?php echo $branch; ?>"><?php echo $branch; ?></option>
					<?php  ?>
				<?php  ?>
	        </select>
	    </div>
        <div class="control-group">
            <label class="control-label branch-label" style="display: inline-block;width: 80px;">锁定分支</label>
            <input type="checkbox" name="locked" value="1" <?php if($locked) echo 'checked';?> />
	    <span id="locked_branch"><?php echo $deploy_branch;?></span>
            <em style="color:red;font-size: 11px;margin-left: 60px;">会限制其他分支push时自动发版流程</em>
        </div>
	    <div class="control-group">
	        <label class="control-label deploy-result" style="display: inline-block;width: 80px;">结果<br/><em style="color:red;font-size: 11px;">刷新看结果</em></label>
	        <textarea type="" class="" style="margin-left: 0px; margin-right: 0px; width: 450px;" heigth="300"><?php echo $deploy_output; ?></textarea>
	    </div>
	    <div class="control-group" style="display: inline-block;margin-left: 80px;">
	    	<input class="deploy-btn" type="submit" value="发版" />
	    	<input class="deploy-btn" type="button" onclick="javascript:document.location.reload()" value="刷新" />
	    </div>
    </form>
    <script>
        function change_branch (that) 
            document.getElementById("locked_branch").innerHTML=that.value;
        
    </script>
</body>
</html>

五、deploy.php 文件权限控制
在有apache的服务器上生成密码htpasswd -bc 文件名（passwd.db） 用户名 密码

或者在线生成http://tool.oschina.net/htpasswd，算法选择MD5 (Apache servers only)
 配置nginx
   location ~ deploy.php 

        auth_basic "Restricted";
        auth_basic_user_file /alidata/server/nginx-1.4.4/conf/passwd.db;
        fastcgi_pass   127.0.0.1:9000;
        fastcgi_index  index.php;
        fastcgi_connect_timeout 200;
        fastcgi_read_timeout 200;
        fastcgi_send_timeout 200;
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 128k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_temp_file_write_size 256k;
        #set_real_ip_from 100.97.0.0/16;
        #real_ip_header X-Forwarded-For;
        fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
        include        fastcgi_params;
    
六、附件
涉及到的其它文件
外网测试服务器上

Cd /root/python
Vim send_email.py
#!/usr/bin/python
# -*- coding:utf-8 -*-
import smtplib,sys,os
from email.mime.text import MIMEText  
import email.mime.multipart
from email.MIMEMultipart import MIMEMultipart
from email.MIMEBase import MIMEBase
from email import Encoders
def send_mail():  
    mailto_list = [xxx@qq.com'] #收件人
    mail_host = "smtp.163.com"  # 设置服务器
    mail_user = "ksx_xxx@163.com"  # 用户名
    mail_pass = "xxxpwd"  # 口令 
    mail_postfix = "163.com"  # 发件箱的后缀
    me = "hello" + "<ksx_xxx@" + mail_postfix + ">"  # 这里的hello可以任意设置，收到信后，将按照设置显示
    content = '可以下载下来进行查看'#邮件正文
    content = content+"<br/>"
    
    fh = open(sys.argv[1])
    for  line in  fh.readlines(): 
    	content = content+line+"<br/>"
    msg = MIMEMultipart()
    body = MIMEText(content, _subtype='html', _charset='utf-8')  # 创建一个实例，这里设置为html格式邮件
    msg.attach(body)
    msg['Subject'] = "项目代码提交记录通知"  # 设置主题
    msg['From'] = me  
    msg['To'] = ";".join(mailto_list)  
    #附件内容，若有多个附件，就添加多个part, 如part1，part2，part3
    part = MIMEBase('application', 'octet-stream')
    # 读入文件内容并格式化，此处文件为当前目录下，也可指定目录 例如：open(r'/tmp/123.txt','rb')
    part.set_payload(open(sys.argv[1],'rb').read())
    Encoders.encode_base64(part)
    ## 设置附件头
    part.add_header('Content-Disposition', 'attachment; filename="diff.txt"')
    msg.attach(part)
      
    try:  
        s = smtplib.SMTP()  
        s.connect(mail_host)  # 连接smtp服务器
        s.login(mail_user, mail_pass)  # 登陆服务器
        s.sendmail(me, mailto_list, msg.as_string())  # 发送邮件
        s.close() 
        os.system("rm -rf %s"%(sys.argv[1])) 
        print "connect success"
        return True  
    except Exception, e:  
        print "connect failed"
        return False
send_mail()
Migrate.php sql同步脚本

<?php

if(php_sapi_name() !== 'cli' && !defined('STDIN')) exit('only run on cli mode');

define("BASEPATH",dirname(__FILE__));

global $db;

include_once(BASEPATH.'/config/database.php');

$master=$db['db_main'];

$master_db=new mysqli($master['hostname'], $master['username'], $master['password'], $master['database']);

if(!$master_db){
	echo 'can not connect mysql'.PHP_EOL;
	exit;
}

mysqli_set_charset($master_db,'UTF8');

$result=mysqli_query($master_db,"SELECT * FROM ".$master['dbprefix']."migration");

if(!$result){

	$sql="CREATE TABLE `".$master['dbprefix']."migration` (`id` INT AUTO_INCREMENT PRIMARY KEY,filename varchar(64) NOT NULL,`version` varchar(64) NOT NULL,`run_time` TIMESTAMP DEFAULT CURRENT_TIMESTAMP) ENGINE=InnoDB DEFAULT CHARSET=utf8;";

	$status=mysqli_query($master_db,$sql);

	$result=mysqli_query($master_db,"SELECT * FROM ".$master['dbprefix']."migration");
}

mysqli_autocommit($master_db,FALSE);


$migrations=array();

while ($row=mysqli_fetch_assoc($result)) {
	
	$migrations[$row['version']]=$row;
}

$versions=array_keys($migrations);

echo 'MIGRATION START'.PHP_EOL;

$php_files = glob(BASEPATH. '/migration/*.php');

foreach ($php_files as $file_path) {

	if (preg_match('/([_a-zA-Z0-9\-]*).php/', basename($file_path),$matches)) {
		
		
		$filename=$matches[1];
		$mcrypt=sha1($matches[1]);

		if(!in_array($mcrypt,$versions)){

			$trans=mysqli_query($master_db,"START TRANSACTION");

			if(!$trans){

				echo "dont support transaction".PHP_EOL;
			}

			global $sql;

			$sql=array();

			include $file_path;

			$success=true;

			foreach ($sql as $k => $v) {

				echo "EXEC SQL : ".$v.PHP_EOL;

				if(mysqli_query($master_db,$v,MYSQLI_USE_RESULT)===FALSE){

					echo "SQL ERROR：".$v.PHP_EOL;

					$success=false;

					mysqli_rollback($master_db);

					break;
				}
			}

			if(!$success) continue;

			if(mysqli_query($master_db,"INSERT INTO ".$master['dbprefix']."migration SET filename='".$filename."',version='".$mcrypt."'",MYSQLI_USE_RESULT)===FALSE){

				echo "update migration error".PHP_EOL;

				mysqli_rollback($master_db);

				continue;
			}

			mysqli_commit($master_db);
		}
	}
}

echo 'MIGRATION END'.PHP_EOL;
