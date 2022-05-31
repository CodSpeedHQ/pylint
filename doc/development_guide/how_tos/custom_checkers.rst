.. _write_a_checker:

How to Write a Checker
======================
You can find some simple examples in the distribution
(`custom.py <https://github.com/PyCQA/pylint/blob/main/examples/custom.py>`_
,
`custom_raw.py <https://github.com/PyCQA/pylint/blob/main/examples/custom_raw.py>`_
and
`deprecation_checker.py <https://github.com/PyCQA/pylint/blob/main/examples/deprecation_checker.py>`_).

.. TODO Create custom_token.py

There are three kinds of checkers:

* Raw checkers, which analyse each module as a raw file stream.
* Token checkers, which analyse a file using the list of tokens that
  represent the source code in the file.
* AST checkers, which work on an AST representation of the module.

The AST representation is provided by the ``astroid`` library.
``astroid`` adds additional information and methods
over ``ast`` in the standard library,
to make tree navigation and code introspection easier.

.. TODO Writing a Raw Checker

.. TODO Writing a Token Checker

Writing an AST Checker
----------------------
Let's implement a checker to make sure that all ``return`` nodes in a function
return a unique constant.
Firstly we will need to fill in some required boilerplate:

.. code-block:: python

  import astroid
  from astroid import nodes
  from typing import TYPE_CHECKING, Optional

  from pylint.checkers import BaseChecker

  if TYPE_CHECKING:
      from pylint.lint import PyLinter


  class UniqueReturnChecker(BaseChecker):

      name = "unique-returns"
      msgs = {
          "W0001": (
              "Returns a non-unique constant.",
              "non-unique-returns",
              "All constants returned in a function should be unique.",
          ),
      }
      options = (
          (
              "ignore-ints",
              {
                  "default": False,
                  "type": "yn",
                  "metavar": "<y or n>",
                  "help": "Allow returning non-unique integers",
              },
          ),
      )


So far we have defined the following required components of our checker:

* A name. The name is used to generate a special configuration
   section for the checker, when options have been provided.

* A message dictionary. Each checker is being used for finding problems
   in your code, the problems being displayed to the user through **messages**.
   The message dictionary should specify what messages the checker is
   going to emit. It has the following format::

       msgs = {
           "message-id": (
               "displayed-message", "message-symbol", "message-help"
           )
       }


   * The ``message-id`` should be a 4-digit number,
     prefixed with a **message category**.
     There are multiple message categories,
     these being ``C``, ``W``, ``E``, ``F``, ``R``,
     standing for ``Convention``, ``Warning``, ``Error``, ``Fatal`` and ``Refactoring``.
     The 4 digits should not conflict with existing checkers
     and the first 2 digits should consistent across the checker.

   * The ``displayed-message`` is used for displaying the message to the user,
     once it is emitted.

   * The ``message-symbol`` is an alias of the message id
     and it can be used wherever the message id can be used.

   * The ``message-help`` is used when calling ``pylint --help-msg``.

We have also defined an optional component of the checker.
The options list defines any user configurable options.
It has the following format::

    options = (
        ("option-symbol", {"argparse-like-kwarg": "value"}),
    )


* The ``option-symbol`` is a unique name for the option.
  This is used on the command line and in config files.
  The hyphen is replaced by an underscore when used in the checker,
  similarly to how you would use  ``argparse.Namespace``.

Next we'll track when we enter and leave a function.

.. code-block:: python

  def __init__(self, linter: Optional["PyLinter"] = None) -> None:
      super().__init__(linter)
      self._function_stack = []

  def visit_functiondef(self, node: nodes.FunctionDef) -> None:
      self._function_stack.append([])

  def leave_functiondef(self, node: nodes.FunctionDef) -> None:
      self._function_stack.pop()

In the constructor we initialise a stack to keep a list of return nodes
for each function.
An AST checker is a visitor, and should implement
``visit_<lowered class name>`` or ``leave_<lowered class name>``
methods for the nodes it's interested in.
In this case we have implemented ``visit_functiondef`` and ``leave_functiondef``
to add a new list of return nodes for this function,
and to remove the list of return nodes when we leave the function.

Finally we'll implement the check.
We will define a ``visit_return`` function,
which is called with an ``.astroid.nodes.Return`` node.

.. _astroid_extract_node:
.. TODO We can shorten/remove this bit once astroid has API docs.

We'll need to be able to figure out what attributes an
``.astroid.nodes.Return` node has available.
We can use ``astroid.extract_node`` for this::

  >>> node = astroid.extract_node("return 5")
  >>> node
  <Return l.1 at 0x7efe62196390>
  >>> help(node)
  >>> node.value
  <Const.int l.1 at 0x7efe62196ef0>

We could also construct a more complete example::

  >>> node_a, node_b = astroid.extract_node("""
  ... def test():
  ...     if True:
  ...         return 5 #@
  ...     return 5 #@
  ... """)
  >>> node_a.value
  <Const.int l.4 at 0x7efe621a74e0>
  >>> node_a.value.value
  5
  >>> node_a.value.value == node_b.value.value
  True

For ``astroid.extract_node``, you can use ``#@`` at the end of a line to choose which statements will be extracted into nodes.

For more information on ``astroid.extract_node``,
see the `astroid documentation <https://pylint.pycqa.org/projects/astroid/en/latest/>`_.

Now we know how to use the astroid node, we can implement our check.

.. code-block:: python

  def visit_return(self, node: nodes.Return) -> None:
      if not isinstance(node.value, nodes.Const):
          return
      for other_return in self._function_stack[-1]:
          if node.value.value == other_return.value.value and not (
              self.config.ignore_ints and node.value.pytype() == int
          ):
              self.add_message("non-unique-returns", node=node)

      self._function_stack[-1].append(node)

Once we have established that the source code has failed our check,
we use ``~.BaseChecker.add_message`` to emit our failure message.

Finally, we need to register the checker with pylint.
Add the ``register`` function to the top level of the file.

.. code-block:: python

  def register(linter: "PyLinter") -> None:
      """This required method auto registers the checker during initialization.
      :param linter: The linter to register the checker to.
      """
      linter.register_checker(UniqueReturnChecker(linter))

We are now ready to debug and test our checker!

Debugging a Checker
-------------------
It is very simple to get to a point where we can use ``pdb``.
We'll need a small test case.
Put the following into a Python file:

.. code-block:: python

  def test():
      if True:
          return 5
      return 5

  def test2():
      if True:
          return 1
      return 5

After inserting pdb into our checker and installing it,
we can run pylint with only our checker::

  $ pylint --load-plugins=my_plugin --disable=all --enable=non-unique-returns test.py
  (Pdb)

Now we can debug our checker!

.. Note::

    ``my_plugin`` refers to a module called ``my_plugin.py``.
    This module can be made available to pylint by putting this
    module's parent directory in your ``PYTHONPATH``
    environment variable or by adding the ``my_plugin.py``
    file to the ``pylint/checkers`` directory if running from source.

Parallelize a Checker
---------------------

``BaseChecker`` has two methods ``get_map_data`` and ``reduce_map_data`` that
permit to parallelize the checks when used with the ``-j`` option. If a checker
actually needs to reduce data it should define ``get_map_data`` as returning
something different than ``None`` and let its ``reduce_map_data`` handle a list
of the types returned by ``get_map_data``.

An example can be seen by looking at ``pylint/checkers/similar.py``.

Testing a Checker
-----------------
Pylint is very well suited to test driven development.
You can implement the template of the checker,
produce all of your test cases and check that they fail,
implement the checker,
then check that all of your test cases work.

Pylint provides a ``pylint.testutils.CheckerTestCase``
to make test cases very simple.
We can use the example code that we used for debugging as our test cases.

.. code-block:: python

  import my_plugin
  import pylint.testutils


  class TestUniqueReturnChecker(pylint.testutils.CheckerTestCase):
      CHECKER_CLASS = my_plugin.UniqueReturnChecker

      def test_finds_non_unique_ints(self):
          func_node, return_node_a, return_node_b = astroid.extract_node("""
          def test(): #@
              if True:
                  return 5 #@
              return 5 #@
          """)

          self.checker.visit_functiondef(func_node)
          self.checker.visit_return(return_node_a)
          with self.assertAddsMessages(
              pylint.testutils.MessageTest(
                  msg_id="non-unique-returns",
                  node=return_node_b,
              ),
          ):
              self.checker.visit_return(return_node_b)

      def test_ignores_unique_ints(self):
          func_node, return_node_a, return_node_b = astroid.extract_node("""
          def test(): #@
              if True:
                  return 1 #@
              return 5 #@
          """)

          with self.assertNoMessages():
              self.checker.visit_functiondef(func_node)
              self.checker.visit_return(return_node_a)
              self.checker.visit_return(return_node_b)


Once again we are using ``astroid.extract_node`` to
construct our test cases.
``pylint.testutils.CheckerTestCase`` has created the linter and checker for us,
we simply simulate a traversal of the AST tree
using the nodes that we are interested in.