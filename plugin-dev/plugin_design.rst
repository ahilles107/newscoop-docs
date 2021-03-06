Plugin Design
====================

How to write your plugin for better integration with Newscoop.

Managing the Plugin Lifecycle
--------------------------------

To manage the plugin from installation to removal, register the following event subscribers: 

  - plugin.install_vendor_plugin_name
  - plugin.remove_vendor_plugin_name
  - plugin update_vendor_plugin_name

``vendor_plugin_name`` is the composer name property (vendor/plugin-name) with slashes `/` and hyphens `-` converted to underscores `_`.

This is an example of an event subscriber class containing placeholder functions for the three events:

.. code-block:: php

    // ExamplePluginBundle/EventListener/LifecycleSubscriber.php
    <?php
    namespace Newscoop\ExamplePluginBundle\EventListener;

    use Symfony\Component\EventDispatcher\EventSubscriberInterface;
    use Newscoop\EventDispatcher\Events\GenericEvent;

    /**
     * Event lifecycle management
     */
    class LifecycleSubscriber implements EventSubscriberInterface
    {
        public function install(GenericEvent $event)
        {
            // do something on install
        }

        public function update(GenericEvent $event)
        {
            // do something on update
        }

        public function remove(GenericEvent $event)
        {
            // do something on remove
        }

        public static function getSubscribedEvents()
        {
            return array(
                'plugin.install.newscoop_example_plugin' => array('install', 1),
                'plugin.update.newscoop_example_plugin' => array('update', 1),
                'plugin.remove.newscoop_example_plugin' => array('remove', 1),
            );
        }
    }

The next step is registering the class in the Event Dispatcher:

.. code-block:: yaml

    // ExamplePluginBundle/Resources/config/services.yml
    services:
        newscoop_example_plugin.lifecyclesubscriber:
            class: Newscoop\ExamplePluginBundle\EventListener\LifecycleSubscriber
            tags:
                - { name: kernel.event_subscriber}

To provide access to all registered container services (``php application/console container:debug``), pass the services as the ``@em`` argument

.. code-block:: yaml

    // ExamplePluginBundle/Resources/config/services.yml
    services:
        newscoop_example_plugin.lifecyclesubscriber:
            class: Newscoop\ExamplePluginBundle\EventListener\LifecycleSubscriber
            arguments:
                - @em
            tags:
                - { name: kernel.event_subscriber}

and use it in your service subscriber:

.. code-block:: php

    // ExamplePluginBundle/EventListener/LifecycleSubscriber.php
    ...
    class LifecycleSubscriber implements EventSubscriberInterface
    {
        private $em;

        public function __construct($em) {
            $this->em = $em;
        }
        ...

..  In subscriber included in this plugin you can find example of database updating (based on doctrine entities and schema tool)


   design/controllers.rst

The Newscoop plugins system is based on the Symfony Bundles system, so almost all Symfony features are available. To create a new controller and route, start by creating the controller class:

.. code-block:: php

        <?php
        // ExamplePluginBundle/Controller/LifecycleSubscriber.php

        namespace Newscoop\ExamplePluginBundle\Controller;

        use Symfony\Bundle\FrameworkBundle\Controller\Controller;
        use Sensio\Bundle\FrameworkExtraBundle\Configuration\Route;
        use Symfony\Component\HttpFoundation\Request;

        class DefaultController extends Controller
        {
            /**
             * @Route("/testnewscoop")
             */
            public function indexAction(Request $request)
            {
                return $this->render('NewscoopExamplePluginBundle:Default:index.html.smarty');
            }
        }
 
Note the annotation for route configuration ``@Route("/testnewscoop")``. Register the controller class in the system:

.. code-block:: yaml

        // ExamplePluginBundle/Resources/config/routing.yml
        newscoop_example_plugin:
            resource: "@NewscoopExamplePluginBundle/Controller/"
            type:     annotation
            prefix:   /

Working with views and templates
+++++++++++++++++++++++++++++++++

The previous Controller example returns a smarty template view:

.. code-block:: php

        return $this->render('NewscoopExamplePluginBundle:Default:index.html.smarty');

You can pass data from the controller to the view:

.. code-block:: php

        return $this->render('NewscoopExamplePluginBundle:Default:index.html.smarty', array(
            'variable' => 'super extra variable'
        ));

The original template is very simple:

.. code-block:: html

        // ExamplePluginBundle/Resources/views/Default/index.html.smarty
        <h1>this is my variable {{ $variable }} !</h1>

For a more complex layout, use the Newscoop default publication theme layout ``page.tpl``:

.. code-block:: html

        // ex. newscoop/themes/publication_1/theme_1/page.tpl
        {{ include file="_tpl/_html-head.tpl" }}
        <div id="wrapper">
            {{ include file="_tpl/header.tpl" }}
            <div id="content" class="clearfix">
                <section class="main entry page">
                    {{ block content }}{{ /block }}
                </section>
                ...
            </div>
        </div>

in the plugin template:

.. code-block:: html

        {{extends file="page.tpl"}}
        {{block content}}
            <h1>this is my variable {{ $variable }} !</h1>
        {{/block}}

Creating Database Entities
---------------------------

Newscoop uses `Doctrine2 <http://www.doctrine-project.org/>`_ for database entity management:

* Get the entity manager from the Newscoop container using ``$this->container->get('em');``
* Use the full FQN notation when getting entities: ``$em->getRepository('Newscoop\ExamplePluginBundle\Entity\OurEntity');``


Adding Admin Controllers
---------------------------------

Admin Controllers consist of an action and a route, as in the example in ``Newscoop\ExamplePluginBundle\Controller\DefaultController``. You can use Twig or Smarty as a template engine. There is information on extending the default admin layout, header, menu and footer in ``Resources/views/Default/admin.html.twig``.

Adding a Plugin Menu to the Newscoop Admin Menu
++++++++++++++++++++++++++++++++++++++++++++++++

The Newscoop Admin menu uses the `KNP Menu Library <https://github.com/KnpLabs/KnpMenu>`_ and `KNP MenuBundle <https://github.com/KnpLabs/KnpMenuBundle>`_. To add a Plugin Menu to the Admin Menu, add the service declaration:

.. code-block:: yaml

    newscoop_example_plugin.configure_menu_listener:
        class: Newscoop\ExamplePluginBundle\EventListener\ConfigureMenuListener
        tags:
          - { name: kernel.event_listener, event: newscoop_newscoop.menu_configure, method: onMenuConfigure }

and the menu configuration listener to your plugin:

.. code-block:: php

        <?php
        // EventListener/ConfigureMenuListener.php
        namespace Newscoop\ExamplePluginBundle\EventListener;

        use Newscoop\NewscoopBundle\Event\ConfigureMenuEvent;

        class ConfigureMenuListener
        {
            public function onMenuConfigure(ConfigureMenuEvent $event)
            {
                $menu = $event->getMenu();
                $menu[getGS('Plugins')]->addChild(
                    'Example Plugin', 
                    array('uri' => $event->getRouter()->generate('newscoop_exampleplugin_default_admin'))
                );
            }
        }

Adding Smarty Template Plugins
-------------------------------

The Newscoop template language is Smarty3. Any Smarty3 plugins in 

``<ExamplePluginBundle>/Resources/smartyPlugins``

are automatically loaded and available in your templates.

Adding Dashboard Widgets
-----------------------------

The Newscoop admin panel automatically loads dashboard widgets from:

``<ExamplePluginBundle>/newscoopWidgets``

Plugin Hooks
---------------------

Plugin hooks let you use existing Newscoop functionality in your plugins. Hooks are defined in PHP files in ``<newscoopRoot>/admin-files/``:

* ``issues/edit.php``
* ``sections/edit.php``
* ``articles/edit_html.php``
* ``system_pref/index.php``
* ``system_pref/do_edit.php``
* ``pub/pub_form.php``

Example hook:

.. code-block:: php

        <?php
        //newscoop/admin-files/articles/edit_html.php:

            echo \Zend_Registry::get('container')->getService('newscoop.plugins.service')
                ->renderPluginHooks('newscoop_admin.interface.article.edit.sidebar', null, array(
                    'article' => $articleObj, 
                    'edit_mode' => $f_edit_mode
                ));
        ?>

..
        //newscoop/admin-files/pub/pub_form.php:
        <?php
            echo \Zend_Registry::get('container')->getService('newscoop.plugins.service')
                ->renderPluginHooks('newscoop_admin.interface.publication.edit', null, array(
                    'publication' => $publicationObj
                ));
        ?>


Adding a Plugin Hook to your Plugin
++++++++++++++++++++++++++++++++++++++++++

Define the hook as a service, an addition to the article editing sidebar ``articles/edit_html.php``:

.. code-block:: yaml

        //Resources/config/services.yml
        newscoop_example_plugin.hooks.listener:
                class:     "Newscoop\ExamplePluginBundle\EventListener\HooksListener"
                arguments: ["@service_container"]
                tags:
                  - { name: kernel.event_listener, event: newscoop_admin.interface.article.edit.sidebar, method: sidebar }

In the ``EventListener`` folder of your plugin directory, ``<ExamplePluginBundle>/EventListener`` create ``HooksListener.php`` as specified in ``services.yml`` above:

.. code-block:: php

        <?php

        namespace Newscoop\ExamplePluginBundle\EventListener;

        use Symfony\Component\HttpFoundation\Request;
        use Newscoop\EventDispatcher\Events\PluginHooksEvent;

        class HooksListener
        {
            private $container;

            public function __construct($container)
            {
                $this->container = $container;
            }

            public function sidebar(PluginHooksEvent $event)
            {
                $response = $this->container->get('templating')->renderResponse(
                    'NewscoopExamplePluginBundle:Hooks:sidebar.html.twig',
                    array(
                        'pluginName' => 'ExamplePluginBundle',
                        'info' => 'This is response from plugin hook!'
                    )
                );

                $event->addHookResponse($response);
            }
        }

The ``sidebar()`` method takes a ``PluginHooksEvent`` type as parameter. The `PluginHooksEvent.php <https://github.com/sourcefabric/Newscoop/blob/master/newscoop/library/Newscoop/EventDispatcher/Events/PluginHooksEvent.php>`_ class collects ``Response`` objects from the plugin admin interface hooks.

Next, inside the ``Resources/views`` directory of your plugin create the ``Hooks`` directory we specified in the HooksListener. Then inside the ``Hooks`` directory create the view for the action: ``sidebar.html.twig``.

.. code-block:: html

        <div class="articlebox" title="{{ pluginName }}">
            <p>{{ info }}</p>
        </div>

The plugin response from the hook shows up in the article editing view:

.. image:: http://i41.tinypic.com/16a1j85.png
