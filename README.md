# Agregar Duracion maxima a express-session
Estos cambios se han realizado a partir de un [pull request](https://github.com/expressjs/session/pull/595) de la libreria (testeado y verificado)

## Cambios en session/cookie.js
Agregar el parametro createdAt:
~~~JS
  get data() {
    return {
      originalMaxAge: this.originalMaxAge,
      partitioned: this.partitioned,
      priority: this.priority
      , expires: this._expires
      , secure: this.secure
      , httpOnly: this.httpOnly
      , domain: this.domain
      , path: this.path
      , sameSite: this.sameSite
      , createdAt: this.createdAt //Este parametro es el que se ha de agregar
    }
  },
~~~

## Cambios en index.js
1. Agregar **@params maxDuration** bajo unset
    ~~~JS
    * @param {String} [options.unset]
    ~~~
    El siguiente codigo
    ~~~JS
    * @param {Numeber} [options.maxDuration] sets the maximum total age in seconds for a session to minimize replay duration
    ~~~

2. Agregar comprobacion **maxDuration** existe bajo el siguiente codigo 
    ~~~JS
    if (secret && !Array.isArray(secret)) {
        secret = [secret];
    }
    ~~~

    Este condicional
    ~~~JS
    if (opts.maxDuration && typeof opts.maxDuration !== "number") {
        throw new TypeError("MaxDuration needs to be specified as a number");
    }
    ~~~
3. Agregar comprobacion **maxDuration** mayor a 0 en la funcion generate 
    ~~~JS
    // generates the new session
    store.generate = function (req) {
        req.sessionID = generateId(req);
        req.session = new Session(req);
        req.session.cookie = new Cookie(cookieOptions);

        if (cookieOptions.secure === 'auto') {
            req.session.cookie.secure = issecure(req, trustProxy);
        }

        if (opts.maxDuration > 0) { //Agregar este if
            req.session.cookie.createdAt = new Date();
        }
    };
    ~~~
4. Agregar function **hasReachedMaxDuration** dentro de \
   `return function session(req, res, next) {`
    ~~~JS
    function hasReachedMaxDuration(sess, opts) {
      if (!opts || !opts.maxDuration || opts.maxDuration <= 0) {
        return false
      }

      if (!sess || !sess.cookie || !sess.cookie.createdAt) {
        debug("session should be timed out, but the created at value is not saved");
        return true
      }

      var createdDate = new Date(sess.cookie.createdAt);
      var nowDate = new Date();

      if ((nowDate.getTime() - createdDate.getTime()) / 1000 < opts.maxDuration) { 
        return false
      }

      return true
    }
    ~~~

5. Agregar condicional (else if) **hasReachedMaxDuration** dentro de
    ~~~JS
    try {
        if (err || !sess) {
          debug('no session found')
          generate()
        }else if(hasReachedMaxDuration(sess, opts)){
          debug('session has reached the max duration')
          generate()
        } else {
          debug('session found')
          inflate(req, sess)
        }
    } catch (e) {
        next(e)
        return
    }
    ~~~
