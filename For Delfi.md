## Backend:
Branch: origin/delfi_server
Path en servidor: /var/www/html/BackMesaSolidaria
Nota:
Desde el servidor se puede hacer pull y despues hacer docker-compose up --build solamente del microservicio que se cambio

## FrontEnd:
Branch: main
Path en servidor: /var/www/mesa-solidaria
Nota:
Copiar el conteniedo de la carpeta dist (se crea al hacer la build) en el path del servidor
#### IMPORTANTE!!!!!
env:
VITE_API_AUTH = '/backend/auth'

VITE_API_MANAGERS = '/backend/api/managers'

VITE_API_NEWS = '/backend/api/news'

VITE_API_PROJECTS = '/backend/api/projects'

VITE_API_VOLUNTARIES = '/backend/api/voluntaries'

## Servidor:
Aplicacion: Filezilla o Termius (Prefiero Termius)
Address: 149.50.146.205
Port: 5043
User: root
Pass: f0A.Cv-uw7yjzC