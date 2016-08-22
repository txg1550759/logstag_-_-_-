# logstag_weixin_email

#!/bin/bash
#logstash _to 报警 微信 邮件by 涂小刚  172826370@qq.com 2016/08/22

以下根据自己的字段加入output 输出即可！
#output {
#        elasticsearch {
#
#            hosts => ["10.171.30.90:9200"]
#            index => "v3-%{+YYYY.MM.dd}"
#    }
#
#
#if [http_resp] == "500" or [http_resp] == "501" or [http_resp] == "502" or [http_resp] == "503" or [http_resp] == "504" {
#
#        exec {
#                command => "echo  '%{timestamp}_http请求错误_后端节点%{xforwardedfor}_域名%{domain}_请求方式%{method}_响应时间%{reqtime}_uri:%{uri}_status_code:%{http_resp}_请求参数%{reqparam}  ' >>/tmp/zabbix_check.txt "
#                }
#        }
#
#}


#下面为微信curl模块和告警解析脚本  放入crontab 1分钟运行一次即可

CropID='wx39db04d'
Secret='kHvngRtf3fvlHxYoOVV4lojPDIC74uat00'
GURL="https://qyapi.weixin.qq.com/cgi-bin/gettoken?corpid=$CropID&corpsecret=$Secret"
Gtoken=$(/usr/bin/curl -s -G $GURL | awk -F\" '{print $4}')

PURL="https://qyapi.weixin.qq.com/cgi-bin/message/send?access_token=$Gtoken"

function body() {
        local int AppID=5                      # 企业号中的应用id
        local UserID="$phone"                        # 部门成员id，zabbix中定义的微信接收者
        local PartyID=""                        # 部门id，定义了范围，组内成员都可接收到消息
        local Msg="$mmsg" # 过滤出zabbix传递的第三个参数
        echo '{'
        echo "\"touser\": \"$UserID\","
        echo "\"toparty\": \"$PartyID\","
        echo "\"msgtype\": \"text\","
        echo "\"agentid\": \" $AppID \","
        echo "\"text\": {"
        echo "\"content\": \"$Msg\""
        echo "},"
        echo "\"safe\":\"0\""
        echo "}"
}


#日志文件路径
logfiles='/tmp/zabbix_check.txt'

#同一个uri 和错误码 超过多少次报警阀值
maxid=15

#uri=`awk '{print $9}' $logfiles`
#取1分钟内值时间
sum_time1=`awk  -F ":" '{print $1":"$2":"$3}' $logfiles |sort |uniq `

#取uri的值 
find_uri=`awk -F "_" '{print $7}' $logfiles |sort |uniq `

check_return () {

                if [ $? -eq "0" ];then
                        echo "$(date +%F-%T) ${name} 成功" >> /data/elk/susc-send-logstash.log                                                                                                                                                                                                         
                else
                        echo " mms $(date +%F-%T) ${name}  error 错误已退出 exiting...."  >> /data/elk/error-send-logstash.log
                fi

}

sed -i '/^$/d' $logfiles

for itime in $sum_time1 ;do
	
	for uri in $find_uri ;do

		for status in 500 501 502 503 504 ;do
			sum_id=`egrep -w "$itime" $logfiles | egrep "$uri" |egrep "status_code:$status" | wc -l`

			if [ "$sum_id" -gt "$maxid" ];then

				mms=`egrep -w "$itime" $logfiles | egrep "$uri" |egrep "status_code:$status" |sort -nr|uniq -c |sort -k1 -nr |head -5 `
	
			   #	echo " $mms ${status} $uri $sum_id 次"
					mmsg=`echo -e "错误码:${status} 1分钟内错误码超过$maxid 前5详情\n$mms\n错误码:${status}\n$uri 总:$sum_id 计数  "`
				#echo " `egrep -qw $i  $logfiles | egrep -q $uri |egrep -q status_code:${status} | wc -l ` "
				#echo " $itme  ${status}  $uri  `egrep -w "$i" $logfiles | egrep "$uri" |egrep status_code:$status | wc -l ` "
				#发送到微信 curl脚本
					for phone in 15507590  13519638 13767 18665024 135792563  152770028 186203397
						do

       						 /usr/bin/curl --data-ascii "$(body )" $PURL

						done
			     #发送到邮件jdk 脚本
				 bash /var/scripts/lib/sendmail.sh 'elk报警'  "${mmsg}"
				 name="发送logstash http_code至微信  $mmsg  "
				 check_return
				 stime=`echo $itime |sed -e 's#\/#\\/#g'`
				 suri=`echo $uri|sed -e 's#\/#\\/#g'`
			         sed -i "s#$stime\(.*\)$suri\(.*\)status_code:$status\(.*\)##g" $logfiles
				

			elif [ "$sum_id" -le "$maxid" ];then
				stime=`echo $itime |sed -e 's#\/#\\/#g'`
				suri=`echo $uri|sed -e 's#\/#\\/#g'`
				sed  -i "s#$stime\(.*\)$suri\(.*\)status_code:$status\(.*\)##g" $logfiles

						
			fi
						
			
		done
	done
done

sed -i '/^$/d' $logfiles
