# Crowdec docker configs

## Delete a decision 

`docker exec -it crowdsec cscli decisions delete --ip 119.221.146.118`

## Inspect alert

refer to : https://docs.crowdsec.net/docs/next/cscli/cscli_alerts_list

`cscli alerts list --ip 1.2.3.4`

`cscli alerts inspect "alert_id" -d`
