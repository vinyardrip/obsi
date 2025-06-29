
```
alias up_reflector='sudo reflector --verbose --country 'Belarus,Russia,Poland,Germany,Ukraine' --protocol https --age 12 --sort rate --latest 20 --save /etc/pacman.d/mirrorlist && echo "--- mirrors update! ---"'
```