# Office 365 監査ログの出力
Office 365 監査ログに関する技術情報についてです。
## Seach-UnifiedAuditLogでのデータ取得最大件数について  
Seach-UnifiedAuditLog で最大何件までログを取れるかという話について、[Docs](https://docs.microsoft.com/ja-jp/powershell/module/exchange/search-unifiedauditlog?view=exchange-ps)を見ると、-SessionCommand RetrunLargeSet など指定すれば、一見すると一回のクエリで最大 50,000 件でのデータが取れるようにみえるかと思います。ただ、試してみると分かりますが、こちらのオプションを付けた所で、-ResultSize に 5,000 を超えた値を指定すれば、エラーになりますし、-ResultSize を指定しないと、100 件しかデータが取れません。結局一回のコマンドで取れるデータについては、-ResultSizeの最大である 5,000 であることは変わりないです。ただ、セッション ID の指定と、-SessionCommand RetrunLargeSet のオプションがあれば、内部的に 50,000 件までの結果を確保しておけるので、繰り返し同じセッション ID でコマンドレットを実行することで、同じ検索条件で、5,000件ずづ順次、10回に分けて、最大50,000件のデータセットが取得できるという動作となります。裏を返せば、50,000件を超えるような広い範囲の検索設定では、同じコマンドレットを複数回実行して、何回かに分けて5,000件ずつログを取得しても、全部のログが取れないという点に注意する必要があります。

## AuditData の扱いについて
Seach-UnifiedAuditLog のコマンドレットでは、Microsoft 365 に関する様々な種類のログが取得可能です。こちらの [Docs](https://docs.microsoft.com/ja-jp/office/office-365-management-api/office-365-management-activity-api-schema#auditlogrecordtype) に取りうるログの種類がまとめられています。ただログの種類によって当然記録されているデータは異なるため、固有の意味のあるデータは、AuditData という列にまとめて JSON 形式で格納されています。そのためログの詳細を CSV 形式などで出力しようとすると、ログの種類(RecordType ごとの Operations) に応じて、JSON の中から取り出す値を個別に指定しなければいけないという煩雑さがあります。今回ここで提示する PowerShell のサンプルでは、一端取り出したログの中から、Operations の種類ごとに 1 つログを取り出してその中に含まれている JSON の属性名を事前に分析しておくことで、個別に属性名を指定することなく、網羅的に CSV 形式にデータを出力しているものとなっています。また今後ニーズが高まるであろう、端末側の操作を抜き出す Endpoint DLP のログを例にとっています(要 M365 E5, M365 E5 Compliance, or Infomatino Protection & Governance ライセンスおよび端末のオンボード)。

# PowerShell スクリプト
## PowerShell を管理権限で立ち上げて事前に一度以下を実行
    Set-ExecutionPolicy -ExecutionPolicy RemoteSigned
    Install-Module -Name ExchangeOnlineManagement

## Endpooint DLP のログをまとめて取得
```
#変数
$User="admin@xxx.onmicrosoft.com"
$Startdate="2021/06/01"
$Enddate="2021/12/31"
$RecordType="DLPEndpoint"
$OutputFolder=[System.Environment]::GetFolderPath("Desktop")+"\"

Import-Module ExchangeOnlineManagement
Connect-IPPSSession -UserPrincipalName $User

#監査ログデータを取得
$sessionId=$RecordType+(Get-Date -Format "yyyyMMdd-HHmm")

#最大 5,000 x 10 回のループでログを取得
$output=@();
for($i = 0; $i -lt 10; $i++){
    $result=Search-UnifiedAuditLog -RecordType $RecordType -Startdate $Startdate -Enddate $Enddate -SessionId $sessionId -SessionCommand ReturnLargeSet -ResultSize 5000
    $output+=$result
    "クエリ"+$i+"回目: "+$result.Count.ToString() + " 件取得"
    if($result.count -ne 5000){break}
}
"合計: "+$Output.Count.ToString() + " 件取得"
    
#Operation の種類ごとに最初の 1 つ目のアイテムから Json に含まれているフィールドを取得
$OperationTypes=$output|Group-Object Operations
$FieldName=@()
foreach($Operation in $OperationTypes){
    $JsonRaw=$Operation.Group[0].AuditData|ConvertFrom-Json
    $FieldsInJson=$JsonRaw|get-member -type NoteProperty
    foreach($f in $FieldsInJson){
      if($FieldName.Contains($f.Name) -eq $false) { $FieldName+=$f.Name}
    }
}

#Select-Object で利用するために、Json をパースする ScriptBlock を生成
$Fields="ResultIndex", "CreationDate","UserIds","Operations","RecordType"
foreach($f in $FieldName){
    $sb=[scriptblock]::Create('$JsonRaw.'+$f)
    if($f -ne "RecordType") {$Fields+=@{Name=$f;Expression=$sb}}
    else {$Fields+=@{Name="RecordType2";Expression=$sb}}
}

#Jsonをパースしながら、CSV 形式に加工
$csv=@();
foreach($row in $output){
    $JsonRaw=$row.AuditData|ConvertFrom-Json
    $data=$row|Select-Object -Property $Fields
    $csv+=$data
 }

#出力
$csv|Export-Csv -Path ($OutputFolder+$sessionId+".csv") -NoTypeInformation -Encoding UTF8

```
