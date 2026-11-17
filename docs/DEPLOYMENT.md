# Deployment

## Status

Deployed and publicly accessible, but not currently in active use: https://aispit.vtsappsteam.edu.rs/login (server of the [Academy of Applied Technical and Preschool Studies Niš](https://international.akademijanis.edu.rs/about-us.html), since September 2026).

## What is established

- Configuration is kept in a separate settings file: database access, the secret key that signs sessions, the key that signs submissions, the maximum upload size, and the upload directory (thesis §4).
- Uploaded files are stored outside the static directory and are served only after an access check (thesis §3.5, §7).
- The application must work in a computer lab without internet access, so all libraries, including the Bootstrap Icons package, are bundled with the project (thesis §1; the icon package is served from `/static/vendor/` by the live site).
- The live application answers with the HTTP header `Server: waitress`, i.e. it runs under the Waitress WSGI server (observed 17 November 2026).

## What is not documented

The thesis does not describe the server set-up. My defence notes mention HTTPS through a Traefik reverse proxy, which cannot be confirmed from outside. The operating system, process management, and my personal role in configuring the server are not documented here.
