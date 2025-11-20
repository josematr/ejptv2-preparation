lo qu edbemos hacer es añadir alñ etc host el sudnominio que tenemos enumerado en mi caso 
10.0.8.14      chaincorp.nyx
ahora vamos a UN BARRIDO CON WFUZZ
└─$ wfuzz -c --hc=404 --hl=11 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -H "Host: FUZZ.chaincorp.nyx" -u 10.0.8.14
ESTE ES EL COMANDO CON ALGUNAS MODIFICACIONES COMO EL 404 PARA OCULTARLO Y LAS LINEAS 11 
 RESULTADO 
                                                                                                                                                                                                      
┌──(kali㉿kali)-[~]
└─$ wfuzz -c --hc=404 --hl=11 -w /usr/share/wordlists/dirbuster/directory-list-lowercase-2.3-medium.txt -H "Host: FUZZ.chaincorp.nyx" -u 10.0.8.14
 /usr/lib/python3/dist-packages/wfuzz/__init__.py:34: UserWarning:Pycurl is not compiled against Openssl. Wfuzz might not work correctly when fuzzing SSL sites. Check Wfuzz's documentation for more information.
********************************************************
* Wfuzz 3.1.0 - The Web Fuzzer                         *
********************************************************

Target: http://10.0.8.14/
Total requests: 207643

=====================================================================
ID           Response   Lines    Word       Chars       Payload                                                                                                                               
=====================================================================

000000001:   400        10 L     35 W       301 Ch      "# directory-list-lowercase-2.3-medium.txt"                                                                                           
000000003:   400        10 L     35 W       301 Ch      "# Copyright 2007 James Fisher"                                                                                                       
000000007:   400        10 L     35 W       301 Ch      "# license, visit http://creativecommons.org/licenses/by-sa/3.0/"                                                                     
000000012:   400        10 L     35 W       301 Ch      "# on atleast 2 different hosts"                                                                                                      
000000006:   400        10 L     35 W       301 Ch      "# Attribution-Share Alike 3.0 License. To view a copy of this"                                                                       
000000011:   400        10 L     35 W       301 Ch      "# Priority ordered case insensative list, where entries were found"                                                                  
000000005:   400        10 L     35 W       301 Ch      "# This work is licensed under the Creative Commons"                                                                                  
000000010:   400        10 L     35 W       301 Ch      "#"                                                                                                                                   
000000008:   400        10 L     35 W       301 Ch      "# or send a letter to Creative Commons, 171 Second Street,"                                                                          
000000009:   400        10 L     35 W       301 Ch      "# Suite 300, San Francisco, California, 94105, USA."                                                                                 
000000013:   400        10 L     35 W       301 Ch      "#"                                                                                                                                   
000000002:   400        10 L     35 W       301 Ch      "#"                                                                                                                                   
000000004:   400        10 L     35 W       301 Ch      "#"                                                                                                                                   
000001897:   400        10 L     35 W       301 Ch      "'"                                                                                                                                   
000002822:   200        20 L     37 W       628 Ch      "utils"                                                                                                                               
000003548:   400        10 L     35 W       301 Ch      "%20"                                                                                                                                 
000004951:   400        10 L     35 W       301 Ch      "$file"        +
como vemos hemos encontado un subdominio utils curioso por lo que vamos a buscarlo en google 
http://utils.chaincorp.nyx/
