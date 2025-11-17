##### - Ruta del perfil de terminal de mac:
`~/.zshrc`

##### - Cómo editar el perfil:
`nano ~/.zshrc`

###### Ejemplo de Alias:
alias ..='cd ..'
alias ...='cd ../..'
alias ll='ls -la'
alias c='clear'

Guardar en nano: Ctrl+O → Enter → Ctrl+X
source ~/.zshrc  # Aplica cambios inmediatos

##### - Donde guardar scripts para poder usarlos en la terminal
~/bin

###### Crear directorio
`mkdir -p ~/bin`

###### Agregar al Path
`echo 'export PATH="$HOME/bin:$PATH"' >> ~/.zshrc`
`source ~/.zshrc`

###### Hacer el script ejecutable
`chmod +x ~/bin/mi_script`

`
