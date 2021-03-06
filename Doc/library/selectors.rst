:mod:`selectors` -- High-level I/O multiplexing
===============================================

.. module:: selectors
   :synopsis: High-level I/O multiplexing.

.. versionadded:: 3.4


Introduction
------------

This module allows high-level and efficient I/O multiplexing, built upon the
:mod:`select` module primitives. Users are encouraged to use this module
instead, unless they want precise control over the OS-level primitives used.

It defines a :class:`BaseSelector` abstract base class, along with several
concrete implementations (:class:`KqueueSelector`, :class:`EpollSelector`...),
that can be used to wait for I/O readiness notification on multiple file
objects. In the following, "file object" refers to any object with a
:meth:`fileno()` method, or a raw file descriptor. See :term:`file object`.

:class:`DefaultSelector` is an alias to the most efficient implementation
available on the current platform: this should be the default choice for most
users.

.. note::
   The type of file objects supported depends on the platform: on Windows,
   sockets are supported, but not pipes, whereas on Unix, both are supported
   (some other types may be supported as well, such as fifos or special file
   devices).

.. seealso::

   :mod:`select`
      Low-level I/O multiplexing module.


Classes
-------

Classes hierarchy::

   BaseSelector
   +-- SelectSelector
   +-- PollSelector
   +-- EpollSelector
   +-- KqueueSelector


In the following, *events* is a bitwise mask indicating which I/O events should
be waited for on a given file object. It can be a combination of the constants
below:

   +-----------------------+-----------------------------------------------+
   | Constant              | Meaning                                       |
   +=======================+===============================================+
   | :const:`EVENT_READ`   | Available for read                            |
   +-----------------------+-----------------------------------------------+
   | :const:`EVENT_WRITE`  | Available for write                           |
   +-----------------------+-----------------------------------------------+


.. class:: SelectorKey

   A :class:`SelectorKey` is a :class:`~collections.namedtuple` used to
   associate a file object to its underlying file decriptor, selected event
   mask and attached data. It is returned by several :class:`BaseSelector`
   methods.

   .. attribute:: fileobj

      File object registered.

   .. attribute:: fd

      Underlying file descriptor.

   .. attribute:: events

      Events that must be waited for this file object.

   .. attribute:: data

      Optional opaque data associated to this file object: for example, this
      could be used to store per-client session.


.. class:: BaseSelector

   A :class:`BaseSelector` is used to wait for I/O event readiness on multiple
   file objects. It supports file stream registration, unregistration, and a
   method to wait for I/O events on those streams, with an optional timeout.
   It's an abstract base class, so cannot be instantiated. Use
   :class:`DefaultSelector` instead, or one of :class:`SelectSelector`,
   :class:`KqueueSelector` etc. if you want to specifically use an
   implementation, and your platform supports it.
   :class:`BaseSelector` and its concrete implementations support the
   :term:`context manager` protocol.

   .. method:: register(fileobj, events, data=None)

      Register a file object for selection, monitoring it for I/O events.

      *fileobj* is the file object to monitor.
      *events* is a bitwise mask of events to monitor.
      *data* is an opaque object.

      This returns a new :class:`SelectorKey` instance, or raises a
      :exc:`ValueError` in case of invalid event mask or file descriptor, or
      :exc:`KeyError` if the file object is already registered.

   .. method:: unregister(fileobj)

      Unregister a file object from selection, removing it from monitoring. A
      file object shall be unregistered prior to being closed.

      *fileobj* must be a file object previously registered.

      This returns the associated :class:`SelectorKey` instance, or raises a
      :exc:`KeyError` if the file object is not registered.

   .. method:: modify(fileobj, events, data=None)

      Change a registered file object monitored events or attached data.

      This is equivalent to :meth:`BaseSelector.unregister(fileobj)` followed
      by :meth:`BaseSelector.register(fileobj, events, data)`, except that it
      can be implemented more efficiently.

      This returns a new :class:`SelectorKey` instance, or raises a
      :exc:`ValueError` in case of invalid event mask or file descriptor, or
      :exc:`KeyError` if the file object is not registered.

   .. method:: select(timeout=None)

      Wait until some registered file objects become ready, or the timeout
      expires.

      If ``timeout > 0``, this specifies the maximum wait time, in seconds.
      If ``timeout <= 0``, the call won't block, and will report the currently
      ready file objects.
      If *timeout* is ``None``, the call will block until a monitored file object
      becomes ready.

      This returns a list of ``(key, events)`` tuple, one for each ready file
      object.

      *key* is the :class:`SelectorKey` instance corresponding to a ready file
      object.
      *events* is a bitmask of events ready on this file object.

   .. method:: close()

      Close the selector.

      This must be called to make sure that any underlying resource is freed.
      The selector shall not be used once it has been closed.

   .. method:: get_key(fileobj)

      Return the key associated to a registered file object.

      This returns the :class:`SelectorKey` instance associated to this file
      object, or raises :exc:`KeyError` if the file object is not registered.


.. class:: DefaultSelector()

   The default selector class, using the most efficient implementation
   available on the current platform. This should be the default choice for
   most users.


.. class:: SelectSelector()

   :func:`select.select`-based selector.


.. class:: PollSelector()

   :func:`select.poll`-based selector.


.. class:: EpollSelector()

   :func:`select.epoll`-based selector.

   .. method:: fileno()

      This returns the file descriptor used by the underlying
      :func:`select.epoll` object.


.. class:: KqueueSelector()

   :func:`select.kqueue`-based selector.

   .. method:: fileno()

      This returns the file descriptor used by the underlying
      :func:`select.kqueue` object.


Examples of selector usage::

   >>> import selectors
   >>> import socket
   >>>
   >>> s = selectors.DefaultSelector()
   >>> r, w = socket.socketpair()
   >>>
   >>> s.register(r, selectors.EVENT_READ)
   SelectorKey(fileobj=<socket.socket fd=4, family=1, type=1, proto=0>, fd=4, events=1, data=None)
   >>> s.register(w, selectors.EVENT_WRITE)
   SelectorKey(fileobj=<socket.socket fd=5, family=1, type=1, proto=0>, fd=5, events=2, data=None)
   >>>
   >>> print(s.select())
   [(SelectorKey(fileobj=<socket.socket fd=5, family=1, type=1, proto=0>, fd=5, events=2, data=None), 2)]
   >>>
   >>> for key, events in s.select():
   ...     if events & selectors.EVENT_WRITE:
   ...         key.fileobj.send(b'spam')
   ...
   4
   >>> for key, events in s.select():
   ...     if events & selectors.EVENT_READ:
   ...         print(key.fileobj.recv(1024))
   ...
   b'spam'
   >>> s.close()
