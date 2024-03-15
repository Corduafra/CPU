# CPU
#script calcolo cpu per ogni utente 

# Imposta la soglia di utilizzo della CPU per il controllo globale
$cpuGlobalThreshold = 70  # Puoi personalizzare questa soglia


function MonitoraggioCaricoCPU {
    # Ottieni il carico totale della CPU
    $cpuLoad = (Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue
    $cpuLoadArrotondato = [Math]::Round($cpuLoad, 2)

    # Controlla se il carico totale della CPU supera la soglia globale
    if ($cpuLoadArrotondato -ge $cpuGlobalThreshold) {
        msg ulj9 "CPU elevata"
        msg administrator "CPU elevata"
    }
}

while ($true) {
    # Attendiamo tot secondi prima di controllare nuovamente
    Start-Sleep -Seconds 180

    # Ottieni il carico totale della CPU
    $cpuLoad = (Get-Counter '\Processor(_Total)\% Processor Time').CounterSamples.CookedValue
    $cpuLoadArrotondato = [Math]::Round($cpuLoad, 2)

    Write-Host "Carico totale della CPU: $cpuLoadArrotondato%"

    # Controlla se il carico totale della CPU supera la soglia globale
    if ($cpuLoadArrotondato -ge $cpuGlobalThreshold) {
        Write-Host "Carico totale della CPU supera la soglia globale ($cpuGlobalThreshold%)."
        msg ulj9 "CPU elevata"
        msg administrator "CPU elevata"
    }

    # Ottieni informazioni sulle sessioni utente RDP utilizzando WMI
    $rdpSessions = quser | ForEach-Object {
        $fields = $_ -split '\s+'
        $sessionId = $fields[2]

        # Controlla se $sessionId contiene un intero
        if ([int]::TryParse($sessionId, [ref]$null)) {
            [PSCustomObject]@{
                UserName = $fields[1]
                idUser = $fields[2]
                LogonTime = $fields[5] + " " +  $fields[6]  # Salva l'orario di accesso
            }
        } else {
            [PSCustomObject]@{
                UserName = $fields[1]
                SessionName = $fields[2]
                idUser = $fields[3]
                LogonTime = $fields[6] + " " + $fields[7]  # Salva l'orario di accesso
            }
        }
    }

    $arrayDati = @()  # Creazione array per la tabella
    $array= @()

    foreach ($session in $rdpSessions) {
        $sessionName = $session.sessionName
        $userName = $session.UserName
        $idUser = $session.idUser

        $currentDateTime = Get-Date  # Orario corrente

        if ($userName -eq "USERNAME") {  # Salta la prima riga di quser 
            continue
        }

        if($logonDateTime -eq ""){
            continue
        }
        
        $logonDateTime = Get-Date $session.LogonTime  # Orario di accesso dell'utente
       

        if ($logonDateTime.Date -ne (Get-Date).Date) {
            # Assegna la data di oggi con l'orario 07:00:00
            $logonDateTime= (Get-Date).Date.AddHours(7)
        }

        # Ottieni informazioni sui processi associati a un utente RDP specifico tramite SI e Id negli user
        function Get-RDPUserProcesses {
            param (
                [int]$idUser
            )

            $userProcesses = @()
            $allProcesses = Get-Process
            foreach ($process in $allProcesses) {
                if ($process.SI -eq $idUser) {
                    $userProcesses += $process
                }
            }
            return $userProcesses
        }
           
        # Calcola l'utilizzo medio della CPU per i processi associati a un utente RDP specifico
        function Get-DifferenzaPerct {
            param (
                [System.Diagnostics.Process[]]$Processes
                
            )

            if ($Processes.Count -eq 0) {
                return 0
            }

            $calcoloCPU = @()  # Array per calcolo CPU deviazione standard 
            $Cpu = 0

            #Tempo tot di CPu per tutti i processi
            $CPUProcesses=@()
            $allProcesses = Get-Process
            $CPU=0
            $Cpu = 0

            for ($i = 1; $i -le 2; $i++) {
                #calcola CPU fino ad ora per det. utente
                foreach ($process in $Processes) {
                    $Cpu += $process.CPU   
                }
                $calcoloCPU += $Cpu

                 #Tempo tot di CPu per tutti i processi
                 foreach ($process in $allProcesses){
                    $CPU += $process.CPU
                }

                $CPUProcesses+= $CPU
                Start-Sleep -Milliseconds 500
            }

            MonitoraggioCaricoCPU

            $CPUProcess = $CPUProcesses[1]-$CPUProcesses[0]
            Write-Host $CPUProcesses
            Write-Host
            
            #variazione della cpu di ogni utente in quell'arco di tempo di 1 secondo
            $variazioneCPU = $calcoloCPU[1] - $calcoloCPU[0]

            Write-Host "calcolo 1: $calcoloCPU[0]"
            Write-Host "calcolo 2: $calcoloCPU[1]"
           
            Write-Host "$CPUProcess "

            Write-Host " dato 1: $variazioneCPU"

            #calcolo scostamento percentuale dal tempo 0 a tempo 1 per vedere se salito o diminuito.
            #-100 serve per vedere se la variazione sia positiva o negativa.
            $Variazione=(($calcoloCPU[1]*100)/$calcoloCPU[0])-100

            
            Write-Host " dato 2: $Variazione"
            
            #se la variazione è negativa per ora salto l'utente
            if($variazioneCPU -gt 0 ){
             $CPUProcess = ($variazioneCPU*100)/$CPUProcess
                
           } elseif ($variazioneCPU -eq 0) {
                <# teoricamente se è uguale a 0 significa che il consume della cpu è stato costante  e possiamo porre i secondi di utilizzo della cpu
                uguale ad uno, perchè per tutto quel secondo abbiamo utilizzato la CPU #>
                $CPUProcess = 0.01/$CPUProcess 
                
            } else{
              
            }

            $CPUProcess=[Math]::Round($CPUProcess,4)

            Write-Host "$userName=$CPUProcess%"
            Write-Host
            Write-Host

            return $CPUProcess
           
        }

         function Get-TimeDifferenceInSeconds {
            param (
                [datetime]$startTime
            )

            $endTime=Get-Date 
            $timeDifference = $endTime.Subtract($startTime)
            $secondsDifference = $timeDifference.TotalSeconds

            $secondsDifference = [Math]::Round($secondsDifference, 0)

            return $secondsDifference
        }

        $secondsDifference = Get-TimeDifferenceInSeconds -startTime $logonDateTime 

        # Calcolo medio CPU utilizzando le due funzioni
        $rdpUserName = $idUser
        $rdpUserProcesses = Get-RDPUserProcesses -idUser $rdpUserName
        $averageCpuUsage = Get-DifferenzaPerct -Processes $rdpUserProcesses 


        # Creazione tabella
        if ($null -ne $averageCpuUsage) {
            if ($logonDateTime -eq (Get-Date).Date.AddHours(7)) {
                $oggettoDati = [PSCustomObject]@{
                    Colonna1 = $userName
                    Colonna2 = "$averageCpuUsage"
                    Colonna5 = $currentDateTime 
                    Colonna3 = $logonDateTime
                    Colonna4 = "non loggato"
                    
                }
            } else {
                $oggettoDati = [PSCustomObject]@{
                    Colonna1 = $userName
                    Colonna2 = "$averageCpuUsage"
                    Colonna5 = $currentDateTime 
                    Colonna3 = $logonDateTime
                    Colonna4 = "loggato"
                }
            }
            $arrayDati += $oggettoDati 
        } 
        else {
            Write-Host "Impossibile ottenere i dati di utilizzo della CPU per l'utente $userName (Sessione ID: $sessionName)."
        }
    }
   
    Write-Output $array | Format-Table -AutoSize

    # Output tabella
    Write-Output $arrayDati | Format-Table -AutoSize


    $quserData = $arrayDati | ForEach-Object {
        $fields = $_ -split '\s+'
        $Username = ($fields[0] -replace '^.*=') -replace ';$'
        $CPU = ($fields[1] -replace '-(?=\d)' -replace '^.*=' -replace ';$') -replace '\.', ','
        $tempo =($fields[3] -replace '^.*=') -replace ';$' 
            [PSCustomObject]@{
                Username = $Username
                CPU = $CPU
                Tempo= $tempo
            }
    }

$CurrentTime = Get-Date -Format "yyyyMMdd-HHmm-TS1"  # Aggiungi "TS1" al nome del file
$OutputFile = Join-Path -Path "C:\checkCpu" -ChildPath "$CurrentTime.txt"

 # Salvare i dati in un file locale
$quserData | Format-Table -AutoSize | Out-File -FilePath $OutputFile

}
