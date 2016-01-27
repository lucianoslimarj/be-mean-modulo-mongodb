# MongoDb - Projeto Final
**Autor:** Luciano Silva de Lima
**Data** Date.now() //em timestamp

## Para qual sistema você usaria o MongoDB (diferente desse)?
```
   Vou mudar a pergunta para: "Para qual sistema você não usaria o MongoDB ?"
   
   Não usaria o mongoDB para aplicações em 2 situações:
   
     1. Aplicações com operações complexas que envolvam vários grupos de dados.
	 2. Aplicações que REALMENTE requerem transações ACID ( Atomicity, Consistency, Isolation e Durability). Não dando alternativa para transações BASE
	    (Basic Availability, Soft-state e Eventual consistency)
		
   Uma tipo de aplicação que se enquadra nesta categoria são as aplicações financeiras, principalmente aquelas envolvendo transações bancárias. Nestas
   aplicações muito provável que uma única operação envolva atualizações em mais de um documento ( contas, movimentações) e o MongoDB não garante a atomicidade
   quando mais de um documento é envolvido.
   Em contra-partida, o mongoDB pode ser usado em uma grande variedade de aplicações: e-commerce, chats, blogs, web analystics, analytics em tempo real, análise de logs de aplicação, etc...
```
## Qual a modelagem da sua coleção de `users`?
```
	users:{
	name: String,
	bio : String,
	registerDate: Date,
	avatarPath  : String,
	settings: {
		backgroundPath: String
	},
	auth: {
		username: String,
		email: String,
		password: String,
		lastAccess: Date,
		online: Boolean,
		disabled: Boolean,
		hashToken: String
	}
	db.users.createIndex( {username: 1},{unique:true} )
	
	--retirar o campo username ? Fazer login pelo email ?
	--Criar indice unico para email e/ou username ?
```
## Qual a modelagem da sua coleção de `projects`?

```
	Considerações: Embora na modelagem, as realocações apresentam-se com sendo 1:1, o documento foi modelado podendo apresentar várias realocações e ainda
    com a flexibilidade de termos mais alguma informação ( além da data ) no processo de realocação.
	
	projects: { 
		name: String,
		description: String,
		startDate: Date,
		dreamDate: Date,
		endDate: Date,
		visible: Boolean, 
		realocate: Boolean,
		expired: Boolean,
		visualizableMod: Boolean,
		members : [{
			user_id: objectId,
			type: String,
			notify: Boolean
		}],
		tags: [String],
		goals: [{
			name: String,
			description: String,
			startDate: Date,
			dreamDate: Date,
			endDate: Date,
			realocate: Boolean,
			expired: Boolean
			tags: [String],
			historics: [{realocateDate: Date}],
			activities: [{activity_id: objectId, name: String}]   // activity_id é um ref para a coleção 'activities'. 'name' é uma desnormalização para facilitar as buscas.
		}]
	}
```
## Qual a modelagem da sua coleção retirada de `projects`?
```	
	Em qualquer modelagem, é importante achar um equilíbrio em performance de leitura, escrita, crescimento de documentos e o nível de consistência de dados, TODOS direcionados pelas características específicas da aplicação. Uma coleção que pode fazer com os documentos excedam o tamanho previamento reservado é a coleção de comentários. Mante-la embutida em documento com alto nivel hierárquico ( objeto dentro de objeto), além de inviabilizar as operações de criação e atualização a partir de documentos pai, pode levar a constantes realocações do documento, causando fragmentações nos arquivos de dados. Ao mesmo tempo, mante-la isolada (sozinha) do restante também não resolveria o problema de criação e atualização, pois os documentos que manteriamos seus IDs não seriam documentos mestres ou filhos de mestres ( Projects[ Goals[ Activities [ [comments_id] ] ]). Devemos também considerar que não temos acessos diretos a comentários sem o contexto das atividades. Mantendo comentários como um array embutido na coleção de atividades, podemos garantir a sua criação a partir desta coleção mestre ao mesmo tempo podemos optar que quando uma atividade for buscada, já tragamos, ou não, alguns comentários. Olhando um pouco para os requisitos de busca de dados, um diz que devemos buscar os projetos juntos com os nomes de suas atividades. De posse disso, para não fazermos múltiplas buscas para atender essa demanda e considerando que o nome de uma atividade não é algo que ficar mudando com frequencia, resolvi criar uma desnormalização de colocar o nome da atividade junto com o seu ID ( como array ), dentro da coleção de goals ( que também é um array de project).
	
		Com isso, a coleção de atividades ficaria assim:
			   
	activities: {
			name: String,
			description: String,
			startDate: Date,
			dreamDate: Date,
			endDate: Date,
			realocate: Boolean,
			expired: Boolean,
			tags: [String],
			historics: [{realocateDate: Date}]
			members : [{
				user_id: objectId,
				type: String,
				notify: Boolean
			}],
			comments:[{
				text: String,
				createDate: Date,
				members : [{
					user_id: objectId,
					type: String,
					notify: Boolean
				}],
				files: [{
					name: String,
					path: String,
					weight: Integer
				}]
			}]
		}
```				
## Create - cadastro

### 1. Cadastre 10 usuários diferentes
```
		//Total de milisegundos em um ano
		var millisInOneYear = 365*24*60*60*1000;
		
		//Retorna uma data aleatória anterior a uma data, até uma certa quantidade de milisegundos
		function getRandomPastDate(data, pastMillis) {
	      	if (pastMillis >= 0) {
				return new Date(data.getTime() - ( _rand() * pastMillis))
			}
			return data;
		}	
	   
	   	//Retorna um boolean randomicamente
		function getRandomBoolean() {
		  return (_rand()* 10) > 5;
		};
	
	   var vetUsuarios = [
		"Francisco Antunes", 
		"Maiara Cordovil", 
		"Alexandre Plastino", 
		"Fernanda Albuquerque", 
		"Elifarley Cruz", 
		"Rodolfo Falante", 
		"Frederico Almeida",
		"Amanda Gentil",
		"Patricia Aranha",
		"Geovana Antonia"
	   ];
	   
	   vetUsuarios.forEach(function(usuario){
	        var posBlank = usuario.indexOf(" ");
			var usu = {
				name: usuario,
				bio:  ((posBlank>=0)?usuario.substring(0,posBlank):usuario) + " bio",
				//registerDate = data atual subtraido de até um ano.
				registerDate: getRandomPastDate(new Date(),millisInOneYear),
				avatarPath: "http://s3.amazonaws.com/avatar/" + usuario.replace(/\s/g,'') + ".jpeg",
				settings: { backgroundPath:"http://s3.amazonaws.com/background/" + usuario.replace(/\s/g,'') + "/"},
				auth: {
					username:  usuario.toLowerCase().replace(/\s/g,''),
					email: usuario.toLowerCase().replace(/\s/g,'.') + "@gmail.com",
					password:"12345",
					lastAccess:null,
					online: getRandomBoolean(),
					disabled: getRandomBoolean(),
					hashToken: "AaYYYyyNOPGaa_"
				},
			};
			//lastAccess = data atual menos uma quantidade de millis aleatória da diferença entre a data corrente e a data do registro do usuário.
			usu.auth.lastAccess = getRandomPastDate( new Date(), (new Date()).getTime() - usu.registerDate.getTime());
			db.users.insert(usu);
	   });
```
> db.users.count()
```
		10
```		
> db.users.findOne()
```
	{
			"_id" : ObjectId("569cd0535114717dd2af2a93"),
			"name" : "Francisco Antunes",
			"bio" : "Francisco bio",
			"registerDate" : ISODate("2015-06-06T17:57:57.778Z"),
			"avatarPath" : "http://s3.amazonaws.com/avatar/FranciscoAntunes.jpeg",
			"settings" : {
					"backgroundPath" : "http://s3.amazonaws.com/background/FranciscoAntunes/"
			},
			"auth" : {
					"username" : "franciscoantunes",
					"email" : "francisco.antunes@gmail.com",
					"password" : "12345",
					"lastAccess" : ISODate("2015-07-30T15:44:04.891Z"),
					"online" : false,
					"disabled" : false,
					"hashToken" : "AaYYYyyNOPGaa_"
			}
	}
```		
> db.users.find({},{"_id":1});
```		
	{ "_id" : ObjectId("569cd0535114717dd2af2a93") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a94") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a95") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a96") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a97") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a98") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a99") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a9a") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a9b") }
	{ "_id" : ObjectId("569cd0535114717dd2af2a9c") }
```	
> db.users.createIndex({"auth.username":1},{unique:true})
```
	{
			"createdCollectionAutomatically" : false,
			"numIndexesBefore" : 1,
			"numIndexesAfter" : 2,
			"ok" : 1
	}
```
> db.users.getIndexes()
```
	[
			{
					"v" : 1,
					"key" : {
							"_id" : 1
					},
					"name" : "_id_",
					"ns" : "be-mean-final.users"
			},
			{
					"v" : 1,
					"unique" : true,
					"key" : {
							"auth.username" : 1
					},
					"name" : "auth.username_1",
					"ns" : "be-mean-final.users"
			}
	]
```
### 2. Cadastre 5 projetos diferentes
	- cada um com 5 membros, sempre diferentes dentro dos projetos;
	- cada um com pelo menos 3 tags diferentes;
		- escolha 1 tag onde deva ficar em 2 projetos;  --(nacional)
		- escolha 1 tag onde deva ficar em 3 projetos;  -- (inovação)
	- cada projeto com pelo menos 1 goal;
		- cada goal com pelo menos 3 tags;
		- cada goal com pelo menos 2 atividades, deixe 1 projeto sem.

```
    // Monta o array de projetos com algumas informações fixas. Após, algumas informações dinâmicas são preenchidas.
    var vetProjects = [
		{ 	name:"Carro movido à água",
			description: "Construção de automóveis que utilizam água como combustível.",
			startDate: null,
			endDate: null,
			dreamDate: null,
			visible: null,
			realocate: null,
			expired: null,
			visualizableMod: null,
			members: [
					{user_id:null, type:"idealizador", notify:null},
					{user_id:null, type:"engenheiro", notify:null},
					{user_id:null, type:"mecanico", notify:null},
					{user_id:null, type:"eletricista", notify:null},
					{user_id:null, type:"montador", notify:null}
			],
			tags:["tranporte","inovação","ciências"],
			goals: [
					{name: "Viabilização do Projeto",
					 description: "Certificar cientificamente que o projeto é víável de acordo com os conhecimentos físicos e mecânicos existentes." ,
					 startDate: null,
					 endDate: null,
			         dreamDate: null,
					 realocate: null,
					 expired: null,
					 tags:["viabilidade","estudo","física"],
					 historics: null,
					 activities:[],
					 pre_activities: [
								  {
								    name: "Montagem do circuito básico",
								    description: "Montar o circuito básico já trará grandes avanços nas evoluções da idéias." ,
								  },
								  {
								    name: "Construção do protótipo",
								    description: "Confeccionar um modelo miniatura do veículo." 
								  }
								 ]
					}
			]
		},
		{ 	name:"Viagem à Marte",
			description: "Visitar para viabilizar habitar Marte.",
			startDate: null,
			endDate: null,
			dreamDate: null,
			visible: null,
			realocate: null,
			expired: null,
			visualizableMod: null,
			members: [
					{user_id:null, type:"idealizador", notify:null},
					{user_id:null, type:"astronauta", notify:null},
					{user_id:null, type:"físico", notify:null},
					{user_id:null, type:"físico", notify:null},
					{user_id:null, type:"físico", notify:null}
			],
			tags:["astronomia","inovação","física"],
			goals: [
					{name: "Pisar no solo de Marte",
					 description: "Um reconhecimento profundo do solo de Marte trará grandes conclusões a respeito do Projeto]." ,
					 startDate: null,
					 endDate: null,
			         dreamDate: null,					 
					 realocate: null,
					 expired: null,
					 tags:["conquista","estudo","solo"],
					 historics: null,
					 activities:[],
					 pre_activities: [
								  {
								    name: "Arrecadação de fundos",
								    description: "Uma reserva generosa proporcionará a aquisição de equipamentos ainda inexistentes." ,
								  },
								  {
								    name: "Convencimento dos astronautas",
								    description: "Engajar os astronautas no projeto, mesmo sabendo que pode ser o último projeto deles." 
								  }
								 ]
					}
			]
		},
		{ 	name:"Combate ao desmatamento na Amazônia",
			description: "O combate ao desmatamento da Amazônia pode amenizar as transformações climáticas em todo Brasil.",
			startDate: null,
			endDate: null,
			dreamDate: null,
			visible: null,
			realocate: null,
			expired: null,
			visualizableMod: null,
			members: [
					{user_id:null, type:"agente do governo", notify:null},
					{user_id:null, type:"controlador", notify:null},
					{user_id:null, type:"agente de segurança", notify:null},
					{user_id:null, type:"agente de segurança", notify:null},
					{user_id:null, type:"piloto", notify:null}
			],
			tags:["polícia","clima","desmatamento"],
			goals: [
					{name: "Reduzir o desmatamento em 40%",
					 description: "Com um controle rígido, identificaremos de imediato os grandes desmatadores." ,
					 startDate: null,
					 endDate: null,
			         dreamDate: null,
					 realocate: null,
					 expired: null,
					 tags:["queimada","controle","redução"],
					 historics: null,
					 activities:[],
					 pre_activities: [
								  {
								    name: "Identificação em tempo real das queimadas",
								    description: "O fragrante nas queimadas será ponto fundamental de encontrar os envolvidos." ,
								  },
								  {
								    name: "Montar grupos de ações territoriais",
								    description: "Uma descentralização nas frentes de ações nos proporcionará um alcance e rapidez nas decisões." 
								  }
								 ]
					}
			]
		},
		{ 	name:"Dessalinização da Água do Mar",
			description: "Com as crise hídricas é de suma importância manter reservas de águas potável.",
			startDate: null,
			endDate: null,
			dreamDate: null,
			visible: null,
			realocate: null,
			expired: null,
			visualizableMod: null,
			members: [
					{user_id:null, type:"financiador", notify:null},
					{user_id:null, type:"químico", notify:null},
					{user_id:null, type:"físico", notify:null},
					{user_id:null, type:"biólogo", notify:null},
					{user_id:null, type:"químico", notify:null}
			],
			tags:["água","sal","inovação"],
			goals: [
					{name: "Atingir produção de 5% do total de água consumida no Brasil",
					 description: "Atingir uma produção equivalente de 5% do consumo total de água a nivel nacional." ,
					 startDate: null,
					 endDate: null,
			         dreamDate: null,
					 realocate: null,
					 expired: null,
					 tags:["consumo","produção","nacional"],
					 historics: null,
					 activities:[],
					 pre_activities: [
								  {
								    name: "Investir em pesquisas para baratear o custo do processo",
								    description: "Atualmente o custo do processo de dessalinização é considerado alto. É preciso reduzir este custo." ,
								  },
								  {
								    name: "Definir processo de liberação dos dejetos do processo",
								    description: "Como o % de água no final do processo é alto, é necessário montar uma infra para devolver esta água ao mar." 
								  }
								 ]
					}
			]
		},
		{ 	name:"Unidade de Polícia Pacificadora",
			description: "Com o aumento das comunidades e o poderio dos marginais, faz-se necessário a retomada do território tomado pela criminalidade.",
			startDate: null,
			endDate: null,
			dreamDate: null,
			visible: null,
			realocate: null,
			expired: null,
			visualizableMod: null,
			members: [
					{user_id:null, type:"financiador", notify:null},
					{user_id:null, type:"agente social", notify:null},
					{user_id:null, type:"policial", notify:null},
					{user_id:null, type:"policial", notify:null},
					{user_id:null, type:"procurador público", notify:null}
			],
			tags:["criminalidade","polícia","nacional"],
			goals: [
					{name: "Retomar o espaço dominado nas 5 maiores comunidades do Brasil",
					 description: "Impedir o avanço das comunidades que mais crescem servirá para dar uma grande baixa no crime organizado." ,
					 startDate: null,
					 endDate: null,
			         dreamDate: null,
					 realocate: null,
					 expired: null,
					 tags:["espaço","reconquista","avanço"],
					 historics: null
					}
			]
		}
		
	];
	
	var firstDayOfYear = new Date((new Date()).getFullYear() + "-01-01");
	var lastDayOfYear = new Date((new Date()).getFullYear() + "-12-31");

	//Retorna uma data randômica entre 2 outras especificadas
	function getRandomDateBetween ( dateInic, dateFin) {
	  var diff = dateFin.getTime() - dateInic.getTime();
	  return new Date(dateInic.getTime() + (_rand() * diff));
	};
	
	//Monta um array com os IDs de usuarios previamente cadastrados para associação como membros de projetos
	var usu_cursor = db.users.find({},{_id:1,name:1});
	var usu_array = usu_cursor.toArray();
	
	//Retorna uma quantidade IDs ( size ) de usuários consecutivos a partir de uma posição especificada
	function getFollowedUserIDs ( fromIndex, size ) {
	    var ret = [];
		if ( fromIndex >= 0 && fromIndex <= size ) {
		  for (var i=fromIndex;i<fromIndex+size;i++) {
			ret.push(usu_array[i]._id);
		  }
		}
		return ret;
	}

	//Percorre do vetor de projetos, previamente montando, enriquecendo cada elemento com informações dinâmicas
	for(var j=0;j<vetProjects.length;j++) {
		var project = vetProjects[j];
		project.startDate = getRandomDateBetween(firstDayOfYear,lastDayOfYear);
		project.endDate = getRandomDateBetween(project.startDate, lastDayOfYear);
		project.dreamDate = getRandomDateBetween(project.startDate, project.endDate);
		project.visible = getRandomBoolean();
		project.realocate = getRandomBoolean();
		project.expired = getRandomBoolean();
		project.visualizableMod = getRandomBoolean();
		
		var vetUserIDs = getFollowedUserIDs (j, 5);
		
		for (var i=0; i<project.members.length; i++) {
		  project.members[i].user_id = vetUserIDs[i];
		  project.members[i].notify = getRandomBoolean();
		}
		project.goals.forEach(function(goal){
			goal.startDate = getRandomDateBetween(project.startDate,project.endDate);
			goal.endDate = getRandomDateBetween(goal.startDate,project.endDate);
			goal.dreamDate = getRandomDateBetween(goal.startDate,goal.endDate);
			goal.realocate = getRandomBoolean();
			goal.expired = getRandomBoolean();
			goal.activities = [];
			if (goal.pre_activities) {
				goal.pre_activities.forEach(function(preactivity){
				    var id = new ObjectId();
				    var activity = {_id: id, name: preactivity.name, description: preactivity.description};
					activity.startDate = getRandomDateBetween(goal.startDate,goal.endDate);
					activity.endDate = getRandomDateBetween(activity.startDate,goal.endDate);
					activity.dreamDate = getRandomDateBetween(activity.startDate,activity.endDate);
					activity.realocate = getRandomBoolean();
					activity.expired = getRandomBoolean();
					activity.tags =  [];
					activity.historics = null;	
					activity.members = [];
					activity.comments = [];
					//Adiciona a atividade
					db.activities.insert(activity);
					goal.activities.push({'activity_id':id, 'name': activity.name});
				});
				//Remove a propriedade "pre_activities" do objeto goal.
				delete goal.pre_activities;
			}
		});
		//Adiciona o projeto
		db.projects.insert(project);	
	}
```	
> db.projects.createIndex({tags:1})
```
{
        "createdCollectionAutomatically" : false,
        "numIndexesBefore" : 1,
        "numIndexesAfter" : 2,
        "ok" : 1
}
```
> db.projects.getIndexes()
```
[
        {
                "v" : 1,
                "key" : {
                        "_id" : 1
                },
                "name" : "_id_",
                "ns" : "be-mean-final.projects"
        },
        {
                "v" : 1,
                "key" : {
                        "tags" : 1
                },
                "name" : "tags_1",
                "ns" : "be-mean-final.projects"
        }
]
```
> db.projects.count()
```
	5
```
> db.projects.findOne()
```
	{
        "_id" : ObjectId("56a7b6005114717dd2af2ad3"),
        "name" : "Carro movido à água",
        "description" : "Construção de automóveis que utilizam água como combustível.",
        "startDate" : ISODate("2016-09-02T05:37:30Z"),
        "endDate" : ISODate("2016-11-06T18:36:05.240Z"),
        "dreamDate" : ISODate("2016-09-12T20:34:16.416Z"),
        "visible" : false,
        "realocate" : false,
        "expired" : false,
        "visualizableMod" : true,
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "idealizador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "engenheiro",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "mecanico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "eletricista",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "montador",
                        "notify" : true
                }
        ],
        "tags" : [
                "tranporte",
                "inovação",
                "ciências"
        ],
        "goals" : [
                {
                        "name" : "Viabilização do Projeto",
                        "description" : "Certificar cientificamente que o projeto é víável de acordo com os conhecimentos físicos e mecânicos existentes.",
                        "startDate" : ISODate("2016-10-24T10:19:21.209Z"),
                        "endDate" : ISODate("2016-10-31T15:52:54.825Z"),
                        "dreamDate" : ISODate("2016-10-29T23:26:51.252Z"),
                        "realocate" : true,
                        "expired" : false,
                        "tags" : [
                                "viabilidade",
                                "estudo",
                                "física"
                        ],
                        "historics" : null,
                        "activities" : [
                                {
                                        "activity_id" : ObjectId("56a7b6005114717dd2af2ad1"),
                                        "name" : "Montagem do circuito básico"
                                },
                                {
                                        "activity_id" : ObjectId("56a7b6005114717dd2af2ad2"),
                                        "name" : "Construção do protótipo"
                                }
                        ]
                }
        ]
}
```
>db.projects.find({}).sort({_id:1}).forEach(function(project){ 
	project.members.forEach(function(member){ 
	   print("project: " + project._id + " -> member: " + member.user_id ); 
   }); 
}) 
```

project: 56a7b6005114717dd2af2ad3 -> member: 569cd0535114717dd2af2a93
project: 56a7b6005114717dd2af2ad3 -> member: 569cd0535114717dd2af2a94
project: 56a7b6005114717dd2af2ad3 -> member: 569cd0535114717dd2af2a95
project: 56a7b6005114717dd2af2ad3 -> member: 569cd0535114717dd2af2a96
project: 56a7b6005114717dd2af2ad3 -> member: 569cd0535114717dd2af2a97
project: 56a7b6005114717dd2af2ad6 -> member: 569cd0535114717dd2af2a94
project: 56a7b6005114717dd2af2ad6 -> member: 569cd0535114717dd2af2a95
project: 56a7b6005114717dd2af2ad6 -> member: 569cd0535114717dd2af2a96
project: 56a7b6005114717dd2af2ad6 -> member: 569cd0535114717dd2af2a97
project: 56a7b6005114717dd2af2ad6 -> member: 569cd0535114717dd2af2a98
project: 56a7b6005114717dd2af2ad9 -> member: 569cd0535114717dd2af2a95
project: 56a7b6005114717dd2af2ad9 -> member: 569cd0535114717dd2af2a96
project: 56a7b6005114717dd2af2ad9 -> member: 569cd0535114717dd2af2a97
project: 56a7b6005114717dd2af2ad9 -> member: 569cd0535114717dd2af2a98
project: 56a7b6005114717dd2af2ad9 -> member: 569cd0535114717dd2af2a99
project: 56a7b6005114717dd2af2adc -> member: 569cd0535114717dd2af2a96
project: 56a7b6005114717dd2af2adc -> member: 569cd0535114717dd2af2a97
project: 56a7b6005114717dd2af2adc -> member: 569cd0535114717dd2af2a98
project: 56a7b6005114717dd2af2adc -> member: 569cd0535114717dd2af2a99
project: 56a7b6005114717dd2af2adc -> member: 569cd0535114717dd2af2a9a
project: 56a7b6005114717dd2af2add -> member: 569cd0535114717dd2af2a97
project: 56a7b6005114717dd2af2add -> member: 569cd0535114717dd2af2a98
project: 56a7b6005114717dd2af2add -> member: 569cd0535114717dd2af2a99
project: 56a7b6005114717dd2af2add -> member: 569cd0535114717dd2af2a9a
project: 56a7b6005114717dd2af2add -> member: 569cd0535114717dd2af2a9b
```

##Retrieve - busca
### 1. Liste as informações dos membros de 1 projeto específico que deve ser buscado pelo seu nome de forma a não ligar para maiúsculas e minúsculas.

> db.projects.find({name:/CArRO MOvido À Água/i},{_id:1,name:1,members:1}).pretty()

```
	{
        "_id" : ObjectId("56a7b6005114717dd2af2ad3"),
        "name" : "Carro movido à água",
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "idealizador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "engenheiro",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "mecanico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "eletricista",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "montador",
                        "notify" : true
                }
        ]
	}
```
### 2. Liste todos os projetos com a tag que você escolheu para os 3 projetos em comum.
> db.projects.find({tags:"inovação"},{name:1,tags:1}).pretty()
```
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad3"),
        "name" : "Carro movido à água",
        "tags" : [
                "tranporte",
                "inovação",
                "ciências"
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad6"),
        "name" : "Viagem à Marte",
        "tags" : [
                "astronomia",
                "inovação",
                "física"
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2adc"),
        "name" : "Dessalinização da Água do Mar",
        "tags" : [
                "água",
                "sal",
                "inovação"
        ]
}
```
### 3. Liste apenas os nomes de todas as atividades para todos os projetos.

> db.projects.find({},{"goals.activities.name":1} ).pretty()
```
	{
			"_id" : ObjectId("56a7b6005114717dd2af2ad3"),
			"goals" : [
					{
							"activities" : [
									{
											"name" : "Montagem do circuito básico"
									},
									{
											"name" : "Construção do protótipo"
									}
							]
					}
			]
	}
	{
			"_id" : ObjectId("56a7b6005114717dd2af2ad6"),
			"goals" : [
					{
							"activities" : [
									{
											"name" : "Arrecadação de fundos"
									},
									{
											"name" : "Convencimento dos astronautas"
									}
							]
					}
			]
	}
	{
			"_id" : ObjectId("56a7b6005114717dd2af2ad9"),
			"goals" : [
					{
							"activities" : [
									{
											"name" : "Identificação em tempo real das queimadas"
									},
									{
											"name" : "Montar grupos de ações territoriais"
									}
							]
					}
			]
	}
	{
			"_id" : ObjectId("56a7b6005114717dd2af2adc"),
			"goals" : [
					{
							"activities" : [
									{
											"name" : "Investir em pesquisas para baratear o custo do processo"
									},
									{
											"name" : "Definir processo de liberação dos dejetos do processo"
									}
							]
					}
			]
	}
	{
			"_id" : ObjectId("56a7b6005114717dd2af2add"),
			"goals" : [
					{
							"activities" : [ ]
					}
			]
	}
```
### 4. Liste todos os projetos que não possuam uma tag.
```
	A tag escolhida foi "polícia". A seguir, buscamos todos os projetos que não têm esse tag. Abaixo, todos que têm.
```
> db.projects.find({tags:{$not:/polícia/}},{_id:1,name:1,tags:1}).pretty()
```
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad3"),
        "name" : "Carro movido à água",
        "tags" : [
                "tranporte",
                "inovação",
                "ciências"
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad6"),
        "name" : "Viagem à Marte",
        "tags" : [
                "astronomia",
                "inovação",
                "física"
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2adc"),
        "name" : "Dessalinização da Água do Mar",
        "tags" : [
                "água",
                "sal",
                "inovação"
        ]
}
```
> db.projects.find({tags:/polícia/},{_id:1,name:1,tags:1}).pretty()
```
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad9"),
        "name" : "Combate ao desmatamento na Amazônia",
        "tags" : [
                "polícia",
                "clima",
                "desmatamento"
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2add"),
        "name" : "Unidade de Polícia Pacificadora",
        "tags" : [
                "criminalidade",
                "polícia",
                "nacional"
        ]
}
```
### 5. Liste todos os usuários que não fazem parte do primeiro projeto cadastrado.
```
	Primeiramente, é montado um array (vetUser1Proj) com os IDs dos membros do primeiro projeto 
	( O primeiro projeto poderia ser buscado pelo nome, mas para exercitar sort, toArray, limit,etc...optou-se pela forma como está).
	Depois, vai-se na coleção de usuários onde os IDs não estão dentre aqueles do array vetUser1Proj.
```
> var vetUsersFirstProj = [];
> db.projects.find({},{_id:1,name:1,"members.user_id":1}).sort({_id:1}).limit(1).toArray()[0].members.forEach(function (member){vetUsersFirstProj.push(member.user_id)});
> vetUsersFirstProj
```
[
        ObjectId("569cd0535114717dd2af2a93"),
        ObjectId("569cd0535114717dd2af2a94"),
        ObjectId("569cd0535114717dd2af2a95"),
        ObjectId("569cd0535114717dd2af2a96"),
        ObjectId("569cd0535114717dd2af2a97")
]
```
> db.users.find({"_id":{$nin:vetUsersFirstProj}},{"_id":1,"name":1})  //Usuários que NÃO estão no primeiro projeto
```
{ "_id" : ObjectId("569cd0535114717dd2af2a98"), "name" : "Rodolfo Falante" }
{ "_id" : ObjectId("569cd0535114717dd2af2a99"), "name" : "Frederico Almeida" }
{ "_id" : ObjectId("569cd0535114717dd2af2a9a"), "name" : "Amanda Gentil" }
{ "_id" : ObjectId("569cd0535114717dd2af2a9b"), "name" : "Patricia Aranha" }
{ "_id" : ObjectId("569cd0535114717dd2af2a9c"), "name" : "Geovana Antonia" }
```
> db.users.find({"_id":{$in:vetUsersFirstProj}},{"_id":1,"name":1})  //Usuários que estão no primeiro projeto
```
{ "_id" : ObjectId("569cd0535114717dd2af2a93"), "name" : "Francisco Antunes" }
{ "_id" : ObjectId("569cd0535114717dd2af2a94"), "name" : "Maiara Cordovil" }
{ "_id" : ObjectId("569cd0535114717dd2af2a95"), "name" : "Alexandre Plastino" }
{ "_id" : ObjectId("569cd0535114717dd2af2a96"), "name" : "Fernanda Albuquerque" }
{ "_id" : ObjectId("569cd0535114717dd2af2a97"), "name" : "Elifarley Cruz" }
```

##Update - alteração

### 1. Adicione para todos os projetos o campo views: 0.
> db.projects.update({},{$set:{views:0}},{multi:true})
```
WriteResult({ "nMatched" : 5, "nUpserted" : 0, "nModified" : 5 })
```
> db.projects.find({},{views:1}).pretty()
```
{ "_id" : ObjectId("56a7b6005114717dd2af2ad3"), "views" : 0 }
{ "_id" : ObjectId("56a7b6005114717dd2af2ad6"), "views" : 0 }
{ "_id" : ObjectId("56a7b6005114717dd2af2ad9"), "views" : 0 }
{ "_id" : ObjectId("56a7b6005114717dd2af2adc"), "views" : 0 }
{ "_id" : ObjectId("56a7b6005114717dd2af2add"), "views" : 0 }
```
### 2. Adicione 1 tag diferente para cada projeto.

> db.projects.find({},{tags:1})
```
{ "_id" : ObjectId("56a7b6005114717dd2af2ad3"), "tags" : [ "tranporte", "inovação", "ciências" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2ad6"), "tags" : [ "astronomia", "inovação", "física" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2ad9"), "tags" : [ "polícia", "clima", "desmatamento" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2adc"), "tags" : [ "água", "sal", "inovação" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2add"), "tags" : [ "criminalidade", "polícia", "nacional" ] }
```
> var vetProjIDs = db.projects.find({},{_id:1}).sort({_id:1}).toArray();
> for(var x=0;x<vetProjIDs.length;x++) {
	db.projects.update({_id:vetProjIDs[x]._id},{$push: { tags: 'project#'+(x+1) }})
}
```
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```
> db.projects.find({},{tags:1})
```
{ "_id" : ObjectId("56a7b6005114717dd2af2ad3"), "tags" : [ "tranporte", "inovação", "ciências", "project#1" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2ad6"), "tags" : [ "astronomia", "inovação", "física", "project#2" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2ad9"), "tags" : [ "polícia", "clima", "desmatamento", "project#3" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2adc"), "tags" : [ "água", "sal", "inovação", "project#4" ] }
{ "_id" : ObjectId("56a7b6005114717dd2af2add"), "tags" : [ "criminalidade", "polícia", "nacional", "project#5" ] }
```
### 3. Adicione 2 membros diferentes para cada projeto.
//Retorna 2 IDs de usuários válidos a partir de indice, considerando os 5 consecutivos já incluídos
//usu_array é uma array com todos os objectIDs da coleção de users.
function getUserIDs ( fromIndex, skip, size ) {
	var ret = [];
	for( var x=0;x<usu_array.length && ret.length<size;x++) {
	  if ( !( x >= fromIndex && x <= (fromIndex + (skip -1) ) ) ) { // é um userID que ainda não foi utilizado
		ret.push(usu_array[x]._id);
	  }
	}
	return ret;
}
>  db.projects.find({},{"members":1}).pretty()
```
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad3"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "idealizador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "engenheiro",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "mecanico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "eletricista",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "montador",
                        "notify" : true
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad6"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "idealizador",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "astronauta",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "físico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "físico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "físico",
                        "notify" : true
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad9"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "agente do governo",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "controlador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "agente de segurança",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "agente de segurança",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "piloto",
                        "notify" : false
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2adc"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "financiador",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "químico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "físico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "biólogo",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a9a"),
                        "type" : "químico",
                        "notify" : false
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2add"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "financiador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "agente social",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "policial",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a9a"),
                        "type" : "policial",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a9b"),
                        "type" : "procurador público",
                        "notify" : false
                }
        ]
}
```
> var vetProjIDs = db.projects.find({},{_id:1}).sort({_id:1}).toArray();
> for(var x=0;x<vetProjIDs.length;x++) {
	var vetUserIDs = getUserIDs(x,5,2);
	if (vetUserIDs) {
		for(var y=0;y<vetUserIDs.length;y++) {
		    var member = { "user_id" : vetUserIDs[y], "type" : "type_" + x + y, "notify" : getRandomBoolean() }
			db.projects.update({ _id:vetProjIDs[x]._id },{ $push: { members: member }});
		}
	}
}
```
WriteResult({ "nMatched" : 1, "nUpserted" : 0, "nModified" : 1 })
```
>  db.projects.find({},{"members":1}).pretty()
```
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad3"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "idealizador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "engenheiro",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "mecanico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "eletricista",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "montador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "type_00",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "type_01",
                        "notify" : false
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad6"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "idealizador",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "astronauta",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "físico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "físico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "físico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "type_10",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "type_11",
                        "notify" : true
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad9"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "agente do governo",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "controlador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "agente de segurança",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "agente de segurança",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "piloto",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "type_20",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "type_21",
                        "notify" : true
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2adc"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "financiador",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "químico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "físico",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "biólogo",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a9a"),
                        "type" : "químico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "type_30",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "type_31",
                        "notify" : true
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2add"),
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "financiador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a98"),
                        "type" : "agente social",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a99"),
                        "type" : "policial",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a9a"),
                        "type" : "policial",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a9b"),
                        "type" : "procurador público",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "type_40",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "type_41",
                        "notify" : false
                }
        ]
}
```
### 4. Adicione 1 comentário em cada atividade, deixe apenas 1 projeto sem.
Ordena-se a coleção de Projects, exibindo o array de activities de goals
 db.projects.find({},{_id:1,"goals.activities":1}).sort({_id:1})
Ao total, são 5 projetos, o ultimo tem uma goal e que ja não tem atividades. Assim, limitarei aos 3 primeiros através limit(3).
Em seguida, chama-se uma função map(...) para eliminar alguma hierarquia no documento retorna
Varre-se essa estrutura e pegando o ID ( refs ) das atividades ( que originalmente estao em goals)
Por último, insere um comentário em cada em cada atividades selecionada.

> db.projects.find({},{_id:1,"goals.activities":1}).sort({_id:1}).limit(3).map(function(elem){return elem.goals}).forEach(function(act){
    for ( var x=0;x<act.length;x++) {
		if (act[x].activities) { 
			act[x].activities.forEach(function(activity){
				var comment = { text: 'comentários para ' + activity.name,
								createDate: new ISODate(),
								members: [],
								files:[]
							  };
				db.activities.update({_id:activity.activity_id},{$push:{comments:comment}});
			});
		}
	}
 });
 > db.activities.find({},{name:1,comments:1}).pretty()
```
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad1"),
        "name" : "Montagem do circuito básico",
        "comments" : [
                {
                        "text" : "comentários para Montagem do circuito básico",
                        "createDate" : ISODate("2016-01-26T21:43:41.265Z"),
                        "members" : [ ],
                        "files" : [ ]
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad2"),
        "name" : "Construção do protótipo",
        "comments" : [
                {
                        "text" : "comentários para Construção do protótipo",
                        "createDate" : ISODate("2016-01-26T21:43:41.274Z"),
                        "members" : [ ],
                        "files" : [ ]
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad4"),
        "name" : "Arrecadação de fundos",
        "comments" : [
                {
                        "text" : "comentários para Arrecadação de fundos",
                        "createDate" : ISODate("2016-01-26T21:43:41.277Z"),
                        "members" : [ ],
                        "files" : [ ]
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad5"),
        "name" : "Convencimento dos astronautas",
        "comments" : [
                {
                        "text" : "comentários para Convencimento dos astronautas",
                        "createDate" : ISODate("2016-01-26T21:43:41.279Z"),
                        "members" : [ ],
                        "files" : [ ]
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad7"),
        "name" : "Identificação em tempo real das queimadas",
        "comments" : [
                {
                        "text" : "comentários para Identificação em tempo real das queimadas",
                        "createDate" : ISODate("2016-01-26T21:43:41.281Z"),
                        "members" : [ ],
                        "files" : [ ]
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ad8"),
        "name" : "Montar grupos de ações territoriais",
        "comments" : [
                {
                        "text" : "comentários para Montar grupos de ações territoriais",
                        "createDate" : ISODate("2016-01-26T21:43:41.283Z"),
                        "members" : [ ],
                        "files" : [ ]
                }
        ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2ada"),
        "name" : "Investir em pesquisas para baratear o custo do processo",
        "comments" : [ ]
}
{
        "_id" : ObjectId("56a7b6005114717dd2af2adb"),
        "name" : "Definir processo de liberação dos dejetos do processo",
        "comments" : [ ]
}
```
### 5. Adicione 1 projeto inteiro com UPSERT.

	var newProject = 	{ 	name: "Inserido via upsert",
							description: "Fazendo parte de um exercício de inserção via upsert.",
							startDate: getRandomDateBetween(firstDayOfYear,lastDayOfYear),
							endDate: getRandomDateBetween(firstDayOfYear,lastDayOfYear),
							dreamDate: getRandomDateBetween(firstDayOfYear,lastDayOfYear),
							visible: getRandomBoolean(),
							realocate: getRandomBoolean(),
							expired: getRandomBoolean(),
							visualizableMod: getRandomBoolean(),
							members: [ {user_id: null, type:"idealizador", notify: null} ],
							tags:["byUPSERT"],
							goals: [
									{	name: "Inserção via upsert deve funcionar",
										description: "Inserção via upsert deve funcionar." ,
										startDate: null,
										endDate: null,
										dreamDate: null,
										realocate: null,
										expired: null,
										tags:["byUPSERT"],
										historics: null,
										activities:null
									}
							]
						};
	// Define os membros
	
	var vetUserIDs = getFollowedUserIDs (0, newProject.members.length);
	for (var i=0; i<newProject.members.length; i++) {
		newProject.members[i].user_id = vetUserIDs[i];
		newProject.members[i].notify = getRandomBoolean();
	}
	// Prepara as goals
	newProject.goals.forEach(function(goal){
		goal.startDate = getRandomDateBetween(newProject.startDate,newProject.endDate);
		goal.endDate = getRandomDateBetween(goal.startDate,newProject.endDate);
		goal.dreamDate = getRandomDateBetween(goal.startDate,goal.endDate);
		goal.realocate = getRandomBoolean();
		goal.expired = getRandomBoolean();
		historics: [];
		goal.activities = [];
	});

	// Realiza a inserção com upsert
    db.projects.update(
		{ name: /Inserido via upsert/i },
		{ $setOnInsert: newProject },
		{ upsert:true }
	);
```
WriteResult({
	"nMatched" : 0,
    "nUpserted" : 1,
    "nModified" : 0,
    "_id" : ObjectId("56a8ebc81e528b675120c07d")
```
> db.projects.find({"_id" : ObjectId("56a8ebc81e528b675120c07d")}).pretty()
```
{
	"_id" : ObjectId("56a8ebc81e528b675120c07d"),
	"name" : "Inserido via upsert",
	"description" : "Fazendo parte de um exercício de inserção via upsert.",
	"startDate" : ISODate("2016-09-01T10:22:37.031Z"),
	"endDate" : ISODate("2016-07-19T04:31:34.921Z"),
	"dreamDate" : ISODate("2016-12-23T19:02:05.683Z"),
	"visible" : false,
	"realocate" : false,
	"expired" : true,
	"visualizableMod" : false,
	"members" : [
			{
					"user_id" : ObjectId("569cd0535114717dd2af2a93"),
					"type" : "idealizador",
					"notify" : true
			}
	],
	"tags" : [
			"byUPSERT"
	],
	"goals" : [
			{
					"name" : "Inserção via upsert deve funcionar",
					"description" : "Inserção via upsert deve funcionar.",
					"startDate" : ISODate("2016-07-30T13:07:05.294Z"),
					"endDate" : ISODate("2016-07-24T15:25:23.968Z"),
					"dreamDate" : ISODate("2016-07-29T20:20:09.545Z"),
					"realocate" : true,
					"expired" : false,
					"tags" : [
							"byUPSERT"
					],
					"historics" : null,
					"activities" : [ ]
			}
	]
}
```
 
--Labs

var vetElements = [];
db.projects.find({},{"goals.activities":1}).toArray().forEach(function(project){
    var vetGoals =  project.goals;
	if (vetGoals) {
		for(var k=0;k<vetGoals.length;k++) {
			var vetActivities = vetGoals[k].activities;
			if (vetActivities) { 
				for ( var x=0;x<vetActivities.length;x++) {
				   vetElements.push({"project":project._id, "activity": "goals." + k + ".activities." + x + ".comments"})
				}
			}
		}
	}
})

vetElements.forEach(function(element){
 var objStr = '{"' + element.activity + '":[]}';
 //var Obj = JSON.parse(objStr);
//printjson(Obj);
 db.projects.update({"_id": element.project},{$set:JSON.parse(objStr)});
})