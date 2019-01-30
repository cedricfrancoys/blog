

some services will be required by almost any request made to the server

services systématiques (besoins bécurrents): pool, auth, access, validate, adapt, orm, report, route

type d'objet : service (singleton) 


// services or providers

class MyProvider extends Singleton {

    protected function __construct() {
    }

    public static function constants() {
        // check for required const
        return ['const1', 'const2'];
    }

}

mandatory methods : 
	getInstance(+dependencies)
	constants()


To instanciate the services, keep track of the instances and deal with dependencies injection, 
services pool = service container (ne peut être surchargé)


	pool->register('service name', 'full class name'); // alternative syntax : accept a map of names/classes

pool->get('service name') 
// if $name == 'pool' : return $this
// check if name matches a registered service
// if already instanciated, get instance
// else tries to inject service as a provider
// returns an instance or null if unknown


possibilté de définir un service alternatif ou de désactiver les comportements liés à un service (en le mettant à null)


comment surcharger ?

1) config locale (config.inc.php) : syntaxe register('auth', 'resiway\auth\AuthenticationManager'); // uses gobal var QN_SERVICES_POOL (as does Service Container class)
2) controller : syntaxe 'providers' => ['auth' => 'resiway\auth\AuthenticationManager']	// requests auth provider AND sets it to custom class


// announce catches providers init and returns an error if something goes wrong
$params = announce ([
	description: '',
	params: [...],
	providers: ['provider1', 'provider2']	// current classes set for those providers, raises an error if not defined
])

list($om, $pm) = [ $params['providers']['ObjectManager'], $params['providers']['PersistentManager']];


try {

}
catch(Exception) {

}


a provider is responsible to check if the constant it uses have been defined

class Singleton {
    protected function __construct() {}

    protected function __clone() {}

    public static function &getInstance() {
        // late-static-bound class name
        $class_name = get_called_class(); 
	$class_id = '__INSTANCE__'.str_replace('\', '_', $class_name);
        if (!isset($GLOBALS[$class_id])) {
            $GLOBALS[$class_id] = call_user_func_array($class_name.'::__construct', func_get_args());
        }
        return $GLOBALS[$class_id];
    }
}

<?php


