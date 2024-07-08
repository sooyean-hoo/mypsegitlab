## Setup GitLab CLI

- install glab
```bash
 brew install glab
```

- setup glab with auths
```bash
glab config set skip_tls_verify true --host gitlab-internal.ps.puppet.net
glab auth login --hostname gitlab-internal.ps.puppet.net
```



## Usage

- locate all the git projects
```bash
FILTERCMD=${FILTERCMD:-tee} # Commands to filter the results first.

find -L . -iname .git -type d | eval ${FILTERCMD} | sort -u 
```

- get all repo
```bash
FILTERCMD=${FILTERCMD:-tee} # Commands to filter the results first.

echo 
echo
echo
GITLAB_HOST=gitlab-internal.ps.puppet.net   \
glab repo list  --all  -P 100000 | eval ${FILTERCMD} 
echo 
echo
echo

```

- create repo
```bash

REPONAME="${REPONAME:-Repo Name}"
REPONAME=`echo "${REPONAME}" | tr -d \(\)\[\]\{\}`

REPOGROUP=${REPOGROUP:-prosvcs-engagements}

REPODESC=${REPODESC:-Created by mypsegitlab.Usage.createrepo }



echo "==GLABCMD==" GITLAB_HOST=gitlab-internal.ps.puppet.net   \
glab repo create --name="${REPONAME}" --group="${REPOGROUP}" --description="${REPODESC}" 

echo 
echo
echo
GITLAB_HOST=gitlab-internal.ps.puppet.net   \
glab repo create --name="${REPONAME}" --group="${REPOGROUP}" --description="${REPODESC}" 
echo 
echo
echo

```


-  Find and Push Repos to GitLab
```bash
DIRHERE=${DIRHERE:-$PWD}
FILTERCMD_LOCALDIR=${FILTERCMD_LOCALDIR:-tee}
FILTERCMD=${FILTERCMD:-tee}

OPS=${OPS:-echo }
if [ "-" = "${OPS}" ] ; then
	OPS="" ;
else
	echo -e "\n=================\nIn simulator mode,  Set OPS=\"-\" to run it for real. Please be on VPN also.\n=================\n"  ;
fi;


OpportunityID=$( defscript=${DIRHERE}/info.md mr show txt OpportunityID | grep -v '==' )
OpportunityNumber=$( defscript=${DIRHERE}/info.md mr show txt OpportunityNumber | grep -v '==' )
OpportunityOwner=$( defscript=${DIRHERE}/info.md mr show txt OpportunityOwner | grep -v '==' )
AccountName=$( defscript=${DIRHERE}/info.md mr show txt AccountName | grep -v '==' )

AccountRegion=$( defscript=${DIRHERE}/info.md mr show txt AccountRegion | grep -v '==' )
BusinessLine=$( defscript=${DIRHERE}/info.md mr show txt BusinessLine | grep -v '==' )
ProjectSOWRequestID=$( defscript=${DIRHERE}/info.md mr show txt ProjectSOWRequestID | grep -v '==' )
PuppetProjectTotalPSUnits=$( defscript=${DIRHERE}/info.md mr show txt PuppetProjectTotalPSUnits | grep -v '==' )
PuppetProjectTotalPSUnits=$( defscript=${DIRHERE}/info.md mr show txt PuppetProjectTotalPSUnits | grep -v '==' )
TotalPSHours=$( defscript=${DIRHERE}/info.md mr show txt TotalPSHours | grep -v '==' )

export GITLAB_HOST=gitlab-internal.ps.puppet.net 


projectsfile=./.listproject.dat

projectsfiledone=./.listproject.done.dat
> $projectsfiledone

echo "===============Ops=Data="
set | grep -E "^DIRHERE|OPS|FILTERCMD"
echo "================Business=Data="
set | grep -E "^Opportunity|AccountName|AccountRegion|BusinessLine|ProjectSOWRequestID|PuppetProjectTotalPSUnits|TotalPSHour|Dates=" | grep -v -E '=$' | grep -v '_='
echo "================="



doneopsfile="./.doneopsfile.dat";
> ${doneopsfile}
while [ -e "${doneopsfile}"   ] ; do  
	echo -e "\n\n================Local Projects Found="
	if [ -s ${doneopsfile} ] ; then
		cat ${doneopsfile} | tee $projectsfile
	else
		FILTERCMD="${FILTERCMD_LOCALDIR}"  defscript=~/Dropbox/puppetshared/bin/mypsegitlab/mypsegitlab.md mr run  locate | grep -v '==='  | sed -E 's/..git//g'  |  ${FILTERCMD} | grep -v -E 'CTRL-C|#' > $projectsfile
		cat $projectsfile ;
	fi;
	
	echo "Halted 10s ..... Cntrl-C to stop"
	sleep 10 ;
	rm -fr  ${doneopsfile}

	cat $projectsfile | while read  pfolder ; do
	
		pushd $PWD
		echo -e "\n\n\n========pfolder=$pfolder========================="
		datainfo=$(  FILTERCMD="grep -i  $(basename $pfolder)"  defscript=~/Dropbox/puppetshared/bin/mypsegitlab/mypsegitlab.md mr run getallrepo | grep -v '==' | grep -v -E 'CTRL-C|#'  )

		echo -e "\t====datainfo=$datainfo===="
		if [ -z "$datainfo" ] ; then


			REPODESC_="Created by mypsegitlab.Usage.createrepo for $( set | grep -E "^Opportunity|AccountName|AccountRegion|BusinessLine|ProjectSOWRequestID|PuppetProjectTotalPSUnits|TotalPSHours|Dates=" | grep -v -E '=$' | grep -v '_=' | tr '[:cntrl:]' ' '  ) "
		

			REPONAME_="${AccountName// /-}-${OpportunityNumber}-$(basename $pfolder)" ;
			
			if [ -z "${OPS}" ] ; then
				pushd $PWD
				cd $pfolder ;

				git checkout main ;
				git config http.sslVerify "false"
				git config https.sslVerify "false"
				
				REPODESC="${REPODESC_}" REPONAME="${REPONAME_}"		REPOGROUP="prosvcs-engagements" defscript=~/Dropbox/puppetshared/bin/mypsegitlab/mypsegitlab.md 	mr run createrepo 
				mr clean ;

				popd
			else
				echo \
				REPODESC="${REPODESC_}" REPONAME="${REPONAME_}"		REPOGROUP="prosvcs-engagements" defscript=~/Dropbox/puppetshared/bin/mypsegitlab/mypsegitlab.md 	mr run createrepo 
			fi;			
			echo  "$pfolder"  >> ${doneopsfile} ;
		else
			data_projectname=`echo ${datainfo} | cut -d' ' -f1 `
			data_giturl=`echo ${datainfo} | cut -d' ' -f2`
	
			for i in `set | grep  'data_' ` ; do 
				echo -e "\t====${i}====" ;
			done ;
			echo 
			echo
			echo
			  
	
			${OPS} cd $pfolder
			${OPS} git config core.sshCommand 'ssh -i ~/.ssh/occkeys'
			${OPS} git remote add psegitlab ${data_giturl}
			${OPS} git status | tee 
			${OPS} git branch --all | tee 
			${OPS} git remote -v 
			${OPS} git push psegitlab --all

		fi;
		echo -e "====================================\n\n\n" ;
		popd ;
		echo ${data_giturl} >> $projectsfiledone
	done
	set | grep  'doneops'
	( [ -e "${doneopsfile}" ]  && echo -e "======MORE OPS====================\n`cat ${doneopsfile}`\n====================" ) || echo -e "======ALL DONE==========\n\n\n" ;
done 

rm -fr ${doneopsfile}
echo 
echo
cat $projectsfiledone | sort -u ;
echo 
echo
cat $projectsfiledone | sort -u | tr ':' '/'   | sed -E 's/git@/https:\/\//g' | sed -E 's/.git$//g'
echo
echo

```