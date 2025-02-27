Making simple Python wrapper kernels
====================================

You can re-use IPython's kernel machinery to easily make new kernels.
This is useful for languages that have Python bindings, such as `Hy
<https://github.com/hylang/hy/>`_ (see
`Calysto Hy <https://github.com/Calysto/calysto_hy/#>`_), or languages
where the REPL can be controlled in a tty using `pexpect <https://pexpect.readthedocs.io/en/latest/>`_,
such as bash.

.. seealso::

   `bash_kernel <https://github.com/takluyver/bash_kernel>`_
     A simple kernel for bash, written using this machinery

The `Metakernel <https://github.com/Calysto/metakernel/#>`_ library makes it easier to
write a wrapper kernel that includes a base set of line and cell magics.  It also has a ``ProcessKernel`` subclass that makes it easy to write kernels that use ``pexpect``.
See `Octave Kernel <https://github.com/Calysto/octave_kernel>`_ as an example.


Required steps
--------------

Subclass :class:`ipykernel.kernelbase.Kernel`, and implement the
following methods and attributes:

.. class:: MyKernel

   .. attribute:: implementation
                  implementation_version
                  banner

     Information for :ref:`msging_kernel_info` replies. 'Implementation' refers
     to the kernel (e.g. IPython), rather than the language (e.g. Python).
     The 'banner' is displayed to the user in console
     UIs before the first prompt. All of these values are strings.

   .. attribute:: language_info

     Language information for :ref:`msging_kernel_info` replies, in a dictionary.
     This should contain the key ``mimetype`` with the mimetype of code in the
     target language (e.g. ``'text/x-python'``), the ``name`` of the language
     being implemented (e.g. ``'python'``), and ``file_extension`` (e.g.
     ``'.py'``).
     It may also contain keys ``codemirror_mode`` and ``pygments_lexer`` if they
     need to differ from :attr:`language`.

     Other keys may be added to this later.

   .. method:: do_execute(code, silent, store_history=True, user_expressions=None, allow_stdin=False)

     Execute user code.

     :param str code: The code to be executed.
     :param bool silent: Whether to display output.
     :param bool store_history: Whether to record this code in history and
         increase the execution count. If silent is True, this is implicitly
         False.
     :param dict user_expressions: Mapping of names to expressions to evaluate
         after the code has run. You can ignore this if you need to.
     :param bool allow_stdin: Whether the frontend can provide input on request
         (e.g. for Python's :func:`raw_input`).

     Your method should return a dict containing the fields described in
     :ref:`execution_results`. To display output, it can send messages
     using :meth:`~ipykernel.kernelbase.Kernel.send_response`. If an error
     occurs during execution, an message of type `error` should be sent
     through :meth:`~ipykernel.kernelbase.Kernel.send_response`
     in addition to an :ref:`execution_results` with an `status` of `error`.
     See :doc:`messaging` for details of the different message types.

.. automethod:: ipykernel.kernelbase.Kernel.send_response

To launch your kernel, add this at the end of your module::

    if __name__ == '__main__':
        from ipykernel.kernelapp import IPKernelApp
        IPKernelApp.launch_instance(kernel_class=MyKernel)

Now create a `JSON kernel spec file <https://jupyter-client.readthedocs.io/en/latest/kernels.html#kernel-specs>`_ and install it using ``jupyter kernelspec install </path/to/kernel>``. Place your kernel module anywhere Python can import it (try current directory for testing). Finally, you can run your kernel using ``jupyter console --kernel <mykernelname>``. Note that ``<mykernelname>`` in the below example is ``echo``.

Example
-------

.. seealso::

    `echo_kernel <https://github.com/jupyter/echo_kernel>`__
      A packaged, installable version of the condensed example below.

``echokernel.py`` will simply echo any input it's given to stdout::

    from ipykernel.kernelbase import Kernel

    class EchoKernel(Kernel):
        implementation = 'Echo'
        implementation_version = '1.0'
        language = 'no-op'
        language_version = '0.1'
        language_info = {
            'name': 'Any text',
            'mimetype': 'text/plain',
            'file_extension': '.txt',
        }
        banner = "Echo kernel - as useful as a parrot"

        def do_execute(self, code, silent, store_history=True, user_expressions=None,
                       allow_stdin=False):
            if not silent:
                stream_content = {'name': 'stdout', 'text': code}
                self.send_response(self.iopub_socket, 'stream', stream_content)

            return {'status': 'ok',
                    # The base class increments the execution count
                    'execution_count': self.execution_count,
                    'payload': [],
                    'user_expressions': {},
                   }

    if __name__ == '__main__':
        from ipykernel.kernelapp import IPKernelApp
        IPKernelApp.launch_instance(kernel_class=EchoKernel)

Here's the Kernel spec ``kernel.json`` file for this::

    {"argv":["python","-m","echokernel", "-f", "{connection_file}"],
     "display_name":"Echo"
    }


Optional steps
--------------

You can override a number of other methods to improve the functionality of your
kernel. All of these methods should return a dictionary as described in the
relevant section of the :doc:`messaging spec <messaging>`.

.. class:: MyCustomKernel

   .. method:: do_complete(code, cursor_pos)

     Code completion

     :param str code: The code already present
     :param int cursor_pos: The position in the code where completion is requested

     .. seealso::

        :ref:`msging_completion` messages

   .. method:: do_inspect(code, cursor_pos, detail_level=0)

     Object introspection

     :param str code: The code
     :param int cursor_pos: The position in the code where introspection is requested
     :param int detail_level: 0 or 1 for more or less detail. In IPython, 1 gets
         the source code.

     .. seealso::

        :ref:`msging_inspection` messages

   .. method:: do_history(hist_access_type, output, raw, session=None, start=None, stop=None, n=None, pattern=None, unique=False)

     History access. Only the relevant parameters for the type of history
     request concerned will be passed, so your method definition must have defaults
     for all the arguments shown with defaults here.

     .. seealso::

        :ref:`msging_history` messages

   .. method:: do_is_complete(code)

     Is code entered in a console-like interface complete and ready to execute,
     or should a continuation prompt be shown?

     :param str code: The code entered so far - possibly multiple lines

     .. seealso::

        :ref:`msging_is_complete` messages

   .. method:: do_shutdown(restart)

     Shutdown the kernel. You only need to handle your own clean up - the kernel
     machinery will take care of cleaning up its own things before stopping.

     :param bool restart: Whether the kernel will be started again afterwards

     .. seealso::

        :ref:`msging_shutdown` messages
