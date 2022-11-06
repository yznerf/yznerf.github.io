---
layout: post
title: > 
    Mailaccounting by script
date: 06. November 2022
tags: [cli, powershell, support]
---

## A robots job

Our customer migrated this year from IBM Notes/Domino to Outlook/Exchange, which - of course - didn't go over without any hickups. 🙄 It ended up to be a very lenthy process, with quite a bit of burocracy, since we had to (and kinda still have to) work with two email-systems on the server and client side.<br>
<br>
While the consultants, who planned the migration, where working on an automated accounting system, leveraging MS Teams and PowerAutomate, we had to be the middle-man in the accounting process, for the time being. A users wish to access a certain mailbox landed in our inbox. To secure that the mail accounting for shared mailboxes was done within the standards of german privacy and data security guidelines, we had to get permission from designated owners of the shared mailbox in question and document it, before we could ask the mail-admins to add the user to the active directory security group, which granted access to the mailbox. <br>
<br>
This involved quite a lot of writing and research on our side, because we had to document, which AD security group gave access to the shared mailboxes and also the security group that contained the "owners", who could grant permission. <br>
If a user asked for access, we had to open a ticket, determine the owner of the mailbox in question, write a mail, askin for permission and if it was granted, route the ticket to mail-administration to add the user to the security group. Finally we could inform the user that the access was granted and how to map the mailbox (auto-mapping for outlook is deactivated for various reasons). <br> 
The administration dumped this repettive, workintensive task on us with no perspective on how long we were supposed to do it this way.
<br>
__This was pain!__ 😠 <br>
<br>
So I decided to outsource some of the process to an interactive script, which I wrote in my little downtime on weekend shifts. It only needs the username and the smtp-adress of the mailbox to which the user wants access to. It checks, if the user already has access and if not sends an email asking for permission to the (selected) owner(s) (I designed an html template with variables) and copies a template, for the text to put into the ticket, to the clipboard. It even saves the query for processing the answer by the owner under the ticket-id and informs the user via mail, after the access is granted by the mail-administration. <br>
<br>
While you save yourself much of the writing with creating mail- and ticket-templates, that doesn't save you the work of manual research - and so many mouseclicks. Writing this script took a little bit more time and effort on my side, but my colleagues (and my superiors) are loving it, and the the introduction of the new accounting system is still only an annoucement, without a hard deadline...
<br>
Meanwhile I wrote some useful functions, that I can use in later scripts and - as always when I'm scripting for a practical application - I've learned a ton.
<br><br>
Moral of the story: If it's a robots job, let the robot do it. 🤖 Even if you have to build the robot yourself.
<br>
Here is a redacted version (for privacy and security reasons) of the script. There are some german bits and variables, because I wrote it in german for my collegues. I've translated as much as reasonably possible, to make it easier for international readers. Some things may be redundant and overall this is not very clean, but it does the job and maybe there are some interesting, useful bits to recycle.

``` Powershell

# Declare global variables
#################################################
# A List of AD-groups for Owners and access, 
# connected to the SMTP adress of the shared Mailbox.
$CsvPath = "$PSScriptRoot\SMB-Liste.csv"
# our internal mailgateway
$SMTPMail = "name of the mailgateway"

#################################################
#                                               #
#                Functions                     #
#################################################

# Validate SamAccountName 
#################################################
Function Get-User {
param(
    [Parameter(Mandatory = $true)]
    [String] $name)
    If($user = get-aduser -Filter "samaccountname -eq '$name'"){
        return $user.SamAccountname
    } 
    Else {
        Write-Host -BackgroundColor Red "Error! SamAccountName not found."
        break
    }
}

# Convert mailbox-selection to object
#################################################
function Select-SMB ($Sel)
{
    $process = $Data | Where-Object {$_.SMTP -eq $Sel.SMTP } 
    return $process
}

# List of AD-groupmebers (-name) from group-name
#################################################
function Grouplist ($GName) # z.B. $Editors/$Owners
{
    $List = Get-ADGroup -filter 'name -eq $GName' | Get-ADGroupMember | Select-Object -ExpandProperty name | sort
    return $List
}


# generate displayname/fullname (GivenName SurName)
#################################################
function Get-Fullname ($SamAcc)
{
    $ADUser = Get-ADUser -Identity $SamAcc
    $Givenname = ($ADUser.GivenName).ToString()
    $Surname = ($ADUser.Surname).ToString()
    $Fullname = $Givenname+" "+$Surname
    return $Fullname
}


# Make a case/process 
#################################################
function Make-Case ($TicketNr, $User, $SMTP, $Editor, $Owner )
{
    If (-not(Test-Path -Path "$PSScriptRoot\$TicketNr\process.csv" -PathType leaf)) {
        Write-Host "A folder for process data will be created:"
        New-Item -Path $PSScriptRoot -Name $Ticketnr -ItemType "directory"
        Start-Sleep 2
        $process = [System.Collections.ArrayList]@()
        $row = "" | Select "SMTP", "Editor",  "Owner", "User"
        $row.SMTP = $SMTP
        $row.Editor = $Editor
        $row.Owner = $Owner
        $row.User = $User
        $process += $row
        $process | Export-Csv -NoTypeInformation -Path "$PSScriptRoot\$TicketNr\process.csv" -Delimiter ";" 
        Write-Host -BackgroundColor Green "The process was saved for later."
    }
    Else{
        Write-Host -BackgroundColor Red "Error: There is already a folder for this Ticket: $TicketNr"
        Write-Host -BackgroundColor Red "If this is an error, please delete it."
        Write-Host ""
        Read-Host "Press ENTER to quit"
        exit
        }
}

# Import process 
#################################################
function Get-Case ($TicketNr)
{
    If (-not(Test-Path -Path "$PSScriptRoot\$TicketNr\process.csv" -PathType Leaf)) {
        Write-Host -BackgroundColor Red "Error: There is no process for this Ticket: $TicketNr"
        Write-Host -BackgroundColor Red "Please check manually or start again"
        Write-Host ""
        Read-Host "Press ENTER to quit"
        exit
    }
    Else{
        $process = Import-Csv -Delimiter ";" -Path "$PSScriptRoot\$TicketNr\process.csv"
        Write-Host ""
        Write-Host "process was imported."
        Write-Host ""
        return $process
        
    }
}

# Test AD-Groupmembership
#################################################
function IsMember ($Group, $User) #z.B $Editor $User
{
    $IsMember = Get-ADGroupMember -Identity (Get-ADGroup -filter 'name -eq $Group') | Where-Object {$_.SamAccountName -eq $User}
        if($IsMember){
            $true
        }
        else{
            $false
        }
}

# Email-Template (redacted)
#################################################

function Email-Head ($Email)
{
    # Head (HTML- und CCS-Stuff copied from a regular mail template)
    #############################################
    Add-Content -Encoding UTF8 $Email "<html><head><meta http-equiv='Content-Type' content='text/html; charset=charset=utf-8'><style type='text/css'>"
    Add-Content -Encoding UTF8 $Email "h4 "
    Add-Content -Encoding UTF8 $Email "    {"
    Add-Content -Encoding UTF8 $Email "        text-align: left;"
    Add-Content -Encoding UTF8 $Email "    }"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "@media screen "
    Add-Content -Encoding UTF8 $Email "{"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "	.headerLineTitle"
    Add-Content -Encoding UTF8 $Email "	{"
    Add-Content -Encoding UTF8 $Email "		width:1.5in;"
    Add-Content -Encoding UTF8 $Email "		display:inline-block;"
    Add-Content -Encoding UTF8 $Email "		margin:0in;"
    Add-Content -Encoding UTF8 $Email "		margin-bottom:.0001pt;"
    Add-Content -Encoding UTF8 $Email "		font-size:11.0pt;"
    Add-Content -Encoding UTF8 $Email "		font-family:'Calibri','sans-serif';"
    Add-Content -Encoding UTF8 $Email "		font-weight:bold;"
    Add-Content -Encoding UTF8 $Email "	}"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "	.headerLineText"
    Add-Content -Encoding UTF8 $Email "	{"
    Add-Content -Encoding UTF8 $Email "		display:inline;"
    Add-Content -Encoding UTF8 $Email "		margin:0in;"
    Add-Content -Encoding UTF8 $Email "		margin-bottom:.0001pt;"
    Add-Content -Encoding UTF8 $Email "		font-size:11.0pt;"
    Add-Content -Encoding UTF8 $Email "		font-family:'Calibri','sans-serif';"
    Add-Content -Encoding UTF8 $Email "		font-weight:normal;"
    Add-Content -Encoding UTF8 $Email "	}"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "   .pageHeader"
    Add-Content -Encoding UTF8 $Email "   {"
    Add-Content -Encoding UTF8 $Email "		font-size:14.0pt;"
    Add-Content -Encoding UTF8 $Email "		font-family:'Calibri','sans-serif';"
    Add-Content -Encoding UTF8 $Email "		font-weight:bold;"
    Add-Content -Encoding UTF8 $Email "		visibility:hidden;"
    Add-Content -Encoding UTF8 $Email "		display:none;"
    Add-Content -Encoding UTF8 $Email "   }   "
    Add-Content -Encoding UTF8 $Email "}"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "@media print "
    Add-Content -Encoding UTF8 $Email "{"
    Add-Content -Encoding UTF8 $Email "	.headerLineTitle"
    Add-Content -Encoding UTF8 $Email "	{"
    Add-Content -Encoding UTF8 $Email "		width:1.5in;"
    Add-Content -Encoding UTF8 $Email "		display:inline-block;"
    Add-Content -Encoding UTF8 $Email "		margin:0in;"
    Add-Content -Encoding UTF8 $Email "		margin-bottom:.0001pt;"
    Add-Content -Encoding UTF8 $Email "		font-size:11.0pt;"
    Add-Content -Encoding UTF8 $Email "		font-family:'Calibri','sans-serif';"
    Add-Content -Encoding UTF8 $Email "		font-weight:bold;"
    Add-Content -Encoding UTF8 $Email "	}"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "	.headerLineText"
    Add-Content -Encoding UTF8 $Email "	{"
    Add-Content -Encoding UTF8 $Email "		display:inline;"
    Add-Content -Encoding UTF8 $Email "		margin:0in;"
    Add-Content -Encoding UTF8 $Email "		margin-bottom:.0001pt;"
    Add-Content -Encoding UTF8 $Email "		font-size:11.0pt;"
    Add-Content -Encoding UTF8 $Email "		font-family:'Calibri','sans-serif';"
    Add-Content -Encoding UTF8 $Email "		font-weight:normal;"
    Add-Content -Encoding UTF8 $Email "	}"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "   .pageHeader"
    Add-Content -Encoding UTF8 $Email "   {"
    Add-Content -Encoding UTF8 $Email "		font-size:14.0pt;"
    Add-Content -Encoding UTF8 $Email "		font-family:'Calibri','sans-serif';"
    Add-Content -Encoding UTF8 $Email "		font-weight:bold;"
    Add-Content -Encoding UTF8 $Email "		visibility:visible;"
    Add-Content -Encoding UTF8 $Email "		display:block;"
    Add-Content -Encoding UTF8 $Email "   }"
    Add-Content -Encoding UTF8 $Email ""
    Add-Content -Encoding UTF8 $Email "}"
    Add-Content -Encoding UTF8 $Email "</style>"
}



function Email-Permission ($Email, $AW, $PF) # -Email $Email -AW $AW -PF $SMTP 
# $Email is a path, $AW a FullName, $PF the SMTP-Adress $SMTP
{
    
    # Email text - permission
    #############################################
    
    # Sorry, can't put that on the internet. Basically HTML formatted text. Added to HTML-File with "Add-content"
}

function Email-UserInfo ($Email, $AW, $PF) 
{

    # E-Mail Text - inform user
    #############################################
    
    # Sorry, can't put that on the internet. Basically HTML formatted text. Added to HTML-File with "Add-content"
}

function Email-Sig ($Email, $me)
{                               
    # Signature
    #############################################
    
    # Sorry, can't put that on the internet. Basically HTML formatted signature, customized with the Fullname of the operator of the script. Added line by line to HTML-File with "Add-content"   
}

# HTML-import and sending email
#################################################

function SendIt ($Email, $To, $Ticket, $SMTP, $SMPTMail)
{
    $EmailBody = (Get-Content -Encoding UTF8 $Email) | Out-String
    Send-MailMessage -Encoding ([System.Text.Encoding]::UTF8) -from 'OUR-Mailbox' -to $To -cc 'OUR-Mailbox' -subject "Access to Mailbox $SMTP [#$Ticket]" -body $EMailBody -BodyAsHtml -smtpserver $SMPTMail
}


#Mailbox-Info
#################################################
function Pf-Info ($SMTP, $Editor, $Editors, $Owner, $Owners)
{
    Write-Host ""
    Write-Host "============="
    Write-Host " Emailadress "
    Write-Host "============="
    Write-Host ""
    Write-Host $SMTP
    Write-Host ""
    Write-Host "================="
    Write-Host "  Editor-Group   "
    Write-Host "================="
    Write-Host ""
    Write-Host "$Editor"
    Write-Host ""
    $Editors
    Write-Host ""
    Write-Host "================"
    Write-Host "  Owner-Group   "
    Write-Host "================"
    Write-Host ""
    Write-Host "$Owner"
    Write-Host ""
    $Owners
    Write-Host ""
}

# Processdepending Templates
#################################################

# Select Owner and generate Displayname
#################################################
function Owner-Fullname ($Owner)
{
    $OList = Get-ADGroup -filter 'name -eq $Owner' | Get-ADGroupMember | Select-Object -Property SamAccountName 
    $OwnerName = $OList | Out-GridView -Title "Select Owner" -OutputMode Single | Select-Object -ExpandProperty SamAccountName
    $ADUser = Get-ADUser $OwnerName
    $Givenname = ($ADUser.GivenName).ToString()
    $Surname = ($ADUser.Surname).ToString()
    $OwnerFullname = $Givenname+" "+$Surname
    return $OwnerFullName
}

function WaGen-SU ($TicketNr, $AW, $SMTP, $Editor, $Owner, $Owners)
{
    # Waiting for permission
    # Ticket text to file and to clipboard
    #############################################
    $SU = "$PSScriptRoot\$TicketNr\Waiting_for_permission.txt"
    
    # Redacted
}


function Mailadmin-text ($TicketNr, $AW, $SMTP, $Editor, $Owner, $OwnerName)
{
    # Routing to mailadmins
    # Ticket text to file and to clipboard
    #############################################
  
  # Redacted  
    
}

##########################################################################
##########################################################################
#                         Here starts the script                         #
##########################################################################
##########################################################################
$me = Get-Fullname -SamAcc $env:USERNAME



# Loop - Script is repeated until the user chooses "no"
#################################################

$Repeat = [System.Management.Automation.Host.ChoiceDescription[]] @("&yes","&no")
while ( $true ) {

    # Selection Menu
    Clear-Host
        Write-Host "================ Mailaccounting ================"
        Write-host ""
        Write-Host "1: SMB-Info"
        Write-Host "2: Get permission from owner"
        Write-Host "3: Permission from owner is available"
        Write-Host "4: Check group-membership and inform user"
        Write-Host "Q: Select 'q' to quit."
        Write-Host ""
        Write-host ""


    # Selection
    #############################################
    $selection = Read-Host "Select the desired option"
    
    
    switch ($selection){

            # Mailbox Info
            #####################################         
         '1' {
            Write-Host ""
            Write-Host "============= Mailbox-Info ============="
            
            $Data = Import-Csv $CsvPath -Delimiter ";" 
            Read-Host "Please press ENTER to select the mailbox"
            $Sel = $Data | Select-Object -Property SMTP  | Out-GridView -Title "Select mailbox" -OutputMode Single 
            
            $process = Select-SMB -Sel $Sel
            
            # spilt object into variables
            $SMTP = $process.SMTP
            $Editor = $process.Editor
            $Owner = $process.Owner
            $Editors = Grouplist -GName $Editor
            $Owners =  Grouplist -GName $Owner
            Pf-Info -SMTP $SMTP -Editor $Editor -Editors $Editors -Owner $Owner -Owners $Owners
         }
         
         # Request permission from owner
         ########################################
         '2' {

            $Data = Import-Csv $CsvPath -Delimiter ";" 
            Read-Host "Please press ENTER to select the mailbox"
            $Sel = $Data | Select-Object -Property SMTP  | Out-GridView -Title "Select mailbox" -OutputMode Single 
            $process = Select-SMB -Sel $Sel
            $SMTP = $process.SMTP
            $Editor = $process.Editor
            $Owner = $process.Owner
            $Editors = Grouplist -GName $Editor
            $Owners =  Grouplist -GName $Owner
            Write-Host ""
            $User = Get-User (Read-Host "Please enter SamAccountName")
            $AW = Get-Fullname -SamAcc $User

            If (IsMember -Group $Editor -User $User){
                Write-Host ""
                Write-Host "$AW is member of the group $Editor"
                Write-Host ""
            }
            ElseIf (IsMember -Group $Owner -User $User){
                Write-Host ""
                Write-Host "$AW is member of the group $Owner"
                Write-Host ""
            }
            Else{
                Write-Host ""
                $Ticket = Read-Host "Please open a ticket and paste the ticketnr"
                Make-Case -TicketNr $Ticket -User $User -SMTP $SMTP -Editor $Editor -Owner $Owner
                $OWAnfrage = "$PSScriptRoot\$Ticket\AnfrageOwner.html"
                Write-Host ""
                Write-Host "Which owners should be contacted?"
                Write-Host ""
                Read-Host "Please press ENTER for selection. (multi-selection possible)"
                $Selection = $Owners | Out-GridView -Title "Select owner" -OutputMode Multiple
                $Mailto = New-Object System.Collections.ArrayList($null)
                    foreach ($i in $Selection)
                    {            
                        $Adresse = (Get-ADUser -Filter 'name -eq $i' -Properties * | Select-Object -ExpandProperty EmailAddress)
                        $null = $Mailto.Add($Adresse)
                    }
                Write-Host ""
                Write-Host "Okay, requesting permission for adding $AW" 
                Write-Host "to AD-Group: $Editor."
                Write-Host "Sending request to the following adresses:"
                Write-Host "$Mailto"
                Email-Head -Email $OWAnfrage
                Email-Permission -Email $OWAnfrage -AW $AW -PF $SMTP
                Email-Sig -Email $OWAnfrage -ich $me
                $Titel = "Everything correct?"
                $message = "Send mail?"
                $yes = New-Object System.Management.Automation.Host.ChoiceDescription "&yes", "yes"
                $no = New-Object System.Management.Automation.Host.ChoiceDescription "&no", "no"
                $options = [System.Management.Automation.Host.ChoiceDescription[]]($yes, $no)
                $choice=$host.ui.PromptForChoice($Titel, $message, $options, 0)
                If ($choice -eq 0){
                    SendIt -Email $OWAnfrage -To $Mailto -Ticket $Ticket -SMTP $SMTP -SMPTMail $SMTPMail
                    Write-Host ""
                    Write-Host "Email sent. Please check our inbox for copy!"
                    Write-Host ""
                    WaGen-SU -TicketNr $Ticket -AW $AW -SMTP $SMTP -Editor $Editor -Owner $Owner -Owners $Owners
                }
                Else{
                    Write-Host ""
                    Write-Host "The process will be aborted. processfolder will be deleted."
                    Write-Host ""
                    Remove-Item -Path "$PSScriptRoot\$Ticket" -Recurse
                    
                    
                }
            }
         
         
         }
          # Permission granted or Owner asked for Editor
          ########################################
         '3' {
            $Titel = "What is the case?"
            $message = "Inquiry by owner (I) or permission for open process (P)?"
            $I = New-Object System.Management.Automation.Host.ChoiceDescription "&Inquiry", "I"
            $P = New-Object System.Management.Automation.Host.ChoiceDescription "&Permission", "P"
            $options = [System.Management.Automation.Host.ChoiceDescription[]]($I, $P)
            $choice=$host.ui.PromptForChoice($Titel, $message, $options, 0)
            If ($choice -eq 0){
                $Data = Import-Csv $CsvPath -Delimiter ";" 
                Read-Host "Press ENTER to select mailbox"
                $Sel = $Data | Select-Object -Property SMTP  | Out-GridView -Title "Select mailbox" -OutputMode Single 
                $process = Select-SMB -Sel $Sel
                $SMTP = $process.SMTP
                $Editor = $process.Editor
                $Owner = $process.Owner
                $Editors = Grouplist -GName $Editor
                $Owners =  Grouplist -GName $Owner
                Write-Host ""
                $User = Get-User (Read-Host "Please enter SamAccountName of user")
                $AW = Get-Fullname -SamAcc $User
                $Ticket = Read-Host "Please open a Ticket and enter Ticketnr."
                Make-Case -TicketNr $Ticket -User $User -SMTP $SMTP -Editor $Editor -Owner $Owner
                
                
                $OwnerName = Owner-Fullname -Owner $Owner
                Mailadmin-text -TicketNr $Ticket -AW $AW -SMTP $SMTP -Editor $Editor -Owner $Owner -OwnerName $OwnerName
            }
            Elseif ($choice -eq 1){
                Write-Host ""
                $Ticket = Read-Host "Please enter Ticketnr. for pending process"
                $process = Get-Case -TicketNr $Ticket
                $SMTP = $process.SMTP
                $Editor = $process.Editor
                $Owner = $process.Owner
                $Editors = Grouplist -GName $Editor
                $Owners =  Grouplist -GName $Owner
                $User = $process.User
                $AW = Get-Fullname -SamAcc $User
                $OwnerName = Owner-Fullname -Owner $Owner
                Mailadmin-text -TicketNr $Ticket -AW $AW -SMTP $SMTP -Editor $Editor -Owner $Owner -OwnerName $OwnerName
            }
         
         }
         # Check group-membership after mail-admins processing
         # optional mail to user
         ########################################
         '4' {
            $Titel = "Check membership"
            $message = "Permission from Mail-Aministration (A) or check membership (M)?"
            $B = New-Object System.Management.Automation.Host.ChoiceDescription "&APermission", "A"
            $M = New-Object System.Management.Automation.Host.ChoiceDescription "&Membership", "M"
            $options = [System.Management.Automation.Host.ChoiceDescription[]]($B, $M)
            $choice1=$host.ui.PromptForChoice($Titel, $message, $options, 0)
                If ($choice1 -eq 0){
                    $Ticket = Read-Host "Please enter Tickernr for open process"
                    $process = Get-Case -TicketNr $Ticket
                    $SMTP = $process.SMTP
                    $Editor = $process.Editor
                    $Owner = $process.Owner
                    $User = $process.User
                    $AW = Get-Fullname $User
                    If (IsMember -Group $Editor -User $User){$Member = 5} 
                    Else {$Member = 6}
                        Switch ($Member) {
                            "5" {
                                    Write-Host ""
                                    Write-Host "$AW is member of group $Editor"
                                    Write-Host ""
                                    $Titel = "Inform User?"
                                    $message = "Send info-mail to user?"
                                    $yes = New-Object System.Management.Automation.Host.ChoiceDescription "&yes", "yes"
                                    $no = New-Object System.Management.Automation.Host.ChoiceDescription "&no", "no"
                                    $options = [System.Management.Automation.Host.ChoiceDescription[]]($yes, $no)
                                    $choice2=$host.ui.PromptForChoice($Titel, $message, $options, 0)
                                        if ($choice2= $yes){
                                                $AW = Get-Fullname -SamAcc $User
                                                $AWInfo = "$PSScriptRoot\$Ticket\AWInfo.html"
                                                $AWMail = (Get-ADUser -Filter 'SamAccountName -eq $User' -Properties * | Select-Object -ExpandProperty EmailAddress)
                                                Email-Head -Email $AWInfo
                                                Email-UserInfo -Email $AWInfo -AW $AW -PF $SMTP
                                                Email-Sig -Email $AWInfo -ich $me
                                                SendIt -Email $AWInfo -To $AWMail -Ticket $Ticket -SMTP $SMTP -SMPTMail $SMTPMail
                                                Kontrolle-SU -Editor $Editor -AW $AW -TicketNr $Ticket
                                                Write-Host -BackgroundColor Green "Mail sent."
                                                Write-Host -BackgroundColor Green "Process completed."
                                                Write-Host -BackgroundColor Green "Deleting process-folder."
                                                Remove-Item "$PSScriptRoot\$Ticket" -Recurse
                                                }
                                        else {Write-Host "See you later!"}
                             }
                            "6" {
                               Write-Host -BackgroundColor Red "Something went wrong. User is not a member. Please check again manually!"
                            }
                        }
                }
                Else {
                    $Data = Import-Csv $CsvPath -Delimiter ";" 
                    Read-Host "Press Enter to select mailbox"
                    $Sel = $Data | Select-Object -Property SMTP  | Out-GridView -Title "Select mailbox" -OutputMode Single 
                    $process = Select-SMB -Sel $Sel
                    $SMTP = $process.SMTP
                    $Editor = $process.Editor
                    $Owner = $process.Owner
                    Write-Host ""
                    $User = Get-User (Read-Host "Please enter SamAccountName of user")
                    $AW = Get-Fullname -SamAcc $User
                     If (IsMember -Group $Editor -User $User){
                         Write-Host ""
                         Write-Host "$AW is already member of $Editor"
                         Write-Host ""
                         $Titel = "Inform user?"
                         $message = "Info-mail to user?"
                         $yes = New-Object System.Management.Automation.Host.ChoiceDescription "&yes", "yes"
                         $no = New-Object System.Management.Automation.Host.ChoiceDescription "&no", "no"
                         $options = [System.Management.Automation.Host.ChoiceDescription[]]($yes, $no)
                         $choice=$host.ui.PromptForChoice($Titel, $message, $options, 0)
                             switch($choice){
                                 '0'{
                                 $Ticket = Read-Host "Please enter Ticketnr."
                                 New-Item -Path "$PSScriptRoot\$Ticket" -ItemType Directory
                                 Start-Sleep 2
                                 $AWMail = (Get-ADUser -Filter 'SamAccountName -eq $User' -Properties * | Select-Object -ExpandProperty EmailAddress)
                                 $AW = Get-Fullname -SamAcc $User
                                 $AWInfo = "$PSScriptRoot\$Ticket\temp.html"
                                 Email-Head -Email $AWInfo
                                 Email-UserInfo -Email $AWInfo -AW $AW -PF $SMTP
                                 Email-Sig -Email $AWInfo -ich $me
                                 SendIt -Email $AWInfo -To $AWMail -Ticket $Ticket -SMTP $SMTP -SMPTMail $SMTPMail
                                 Kontrolle-SU -Editor $Editor -AW $AW -TicketNr $Ticket
                                 Remove-Item "$PSScriptRoot\$Ticket" -Recurse                                 
                                 }
                                 '1'{Write-Host "Okay. No infomail"}
                             }
                     }
                     ElseIf (IsMember -Group $Owner -User $User){
                         Write-Host ""
                         Write-Host "$AW is member of the $Owner group"
                         Write-Host ""
            
                     }
                     Else{
                         Write-Host "$AW has not access to $SMTP yet!"
                     }
                }
            }
            
         
         'q' {exit}
    }
 
 $choice4 = $Host.UI.PromptForChoice("Done! Again?","",$Repeat,0)
  if ( $choice4 -ne 0 ) {break}
  }
````