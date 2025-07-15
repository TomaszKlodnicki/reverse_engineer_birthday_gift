```
 _     _      _   _         _             
| |__ (_)_ __| |_| |__   __| | __ _ _   _ 
| '_ \| | '__| __| '_ \ / _` |/ _` | | | |
| |_) | | |  | |_| | | | (_| | (_| | |_| |
|_.__/|_|_|   \__|_| |_|\__,_|\__,_|\__, |
                                    |___/ 
      _           _ _                       
  ___| |__   __ _| | | ___ _ __   __ _  ___ 
 / __| '_ \ / _` | | |/ _ \ '_ \ / _` |/ _ \
| (__| | | | (_| | | |  __/ | | | (_| |  __/
 \___|_| |_|\__,_|_|_|\___|_| |_|\__, |\___|
                                 |___/      
```

## About

I received this program as a birthday gift from a friend of mine. The challenge is to reverse engineer it and uncover the hidden password. The program was created by [Nefretus](https://github.com/Nefretus). Thanks!

## Setp 1

```bash
strings challange > string.txt
```

Let’s see the hardcoded strings inside this file. There’s no password for sure, but what do we have here?

```
przekaz haslo jako pierwszy command line argument 
Sza sza sza!!!
No niestety byku
```

Now we now what we need to get as output sza sza sza!!! :D

```
GCC: (Ubuntu 11.4.0-1ubuntu1~22.04) 11.4.0
```

And it was compiled on ubuntu.

## Step 2

To get info about build arch lets exectue `file` command

```bash
file challenge > file.txt
```

Now we know that it's for `x86` and there is no debug info commpiled inside.

## Step 3

Let's go to `Ghidra`.

![import fail](ghidra_import_fail.png "fail to import required libs")

I encountered an issue loading libraries from the x86 architecture because I’m using an Apple ARM machine. To resolve this, I Located the missing libraries on my Linux x86 machine, copied them to ./libs/ and imported them into Ghidra, which resolved the errors.

![import success](resolve_ok.png "import success")

Next, I searched for the entry point `libc.so.6::__libc_start_main` and locate `main()` function.

After finding main i marked program input, and stared labeling variables. 

I found argument count check
```cpp
  if (argc < 2) {
    uVar2 = cout(&err_stream,"przekaz haslo jako pierwszy command line argument");
    _ZNSolsEPFRSoS_E(uVar2,_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_);
    main_return = 5;
  }
```

Then I've labeled string check function:

```cpp
    bVar1 = check_pass_func((char *)&hidden_pass,input_tab[1]); // if return true hidden_pass == input_string
    if (bVar1) {
      uVar2 = cout(&out_stream,"Sza sza sza!!!");
      _ZNSolsEPFRSoS_E(uVar2,_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_);
    }
    else {
      uVar2 = cout(&out_stream,"No niestety byku");
      _ZNSolsEPFRSoS_E(uVar2,_ZSt4endlIcSt11char_traitsIcEERSt13basic_ostreamIT_T0_ES6_);
    }
```
I've also found function where this hidden password is generated

```cpp
    create_string(&hidden_pass);
    create_strgin_from_data((char *)&string_one,&string_one_data,6,(char *)&hidden_pass); // here we are loading data of len 6, to some array
    _ZNSaIcED1Ev(&hidden_pass); // some kind of free after move
    create_string(&hidden_pass);
    create_strgin_from_data((char *)&string_two,&string_two_data,0x55,(char *)&hidden_pass); // here we are loading data of len 0x55, to some array
    _ZNSaIcED1Ev(&hidden_pass);
    edit_pass((char *)&hidden_pass,(char *)&string_two,(char *)&string_one); // generate function
```

Here is decompiled password generating section:

```cpp
  for (iterator = 0; (&string_two_data)[iterator] != '\0'; iterator = iterator + 1) {
    temp_char_ptr = (char *)_ZNKSsixEj(str_one,iterator);
    temp_char = *temp_char_ptr;
    string_two_len = string_len(str_two);
    pbVar1 = (char *)_ZNKSsixEj(str_two,iterator % string_two_len);
    _ZNSs9push_backEc(&temp_str,(int)(char)(temp_char ^ *pbVar1));
  }
```

`string_two` and `str_one` are bytes from which password is created. The function `_ZNKSsixEj` appears to be some kind of `operator[]`.

```cpp
{0x18, 0xd6, 0x15, 0xca, 0xfa, 0x77} //string one data
```

## Step 4

To get the password, we need to recreate the code logic. I've copied the bytes from `string_one` and `string_two` into C-style arrays and implemented this `for` loop. The password encoder logic is in `resolve.cpp`.

```cpp
  for (int i = 0; i < sizeof(array_two); i++) {

    uint8_t temp_char = array_one[i % sizeof(array_one)];
    size_t string_two_len = sizeof(array_two);

    uint8_t new_char = array_two[i % string_two_len];
    pass += static_cast<char>(temp_char ^ new_char);
  }
```

Let's gooooo!

``` bash
g++ resolve.cpp resolve
./resolve
ROZPYKANE_PRZEZ_ZAWODOWCA_GRATULACJE_SZEFIE_NAGRODA_W_NAJWYZSZEJ_POLCE_NA_WARSZTACIE
./challenge ROZPYKANE_PRZEZ_ZAWODOWCA_GRATULACJE_SZEFIE_NAGRODA_W_NAJWYZSZEJ_POLCE_NA_WARSZTACIE
Sza sza sza!!!
```