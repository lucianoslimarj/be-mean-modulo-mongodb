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
			historics: [{realocateDate: Date}]
			activities: [{
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
				}]
			}]
		}]
	}
```
## Qual a modelagem da sua coleção retirada de `projects`?
```	
	A coleção candidata a não ser um documento embutido na coleção de projetos é a coleção de 'comments'. Dois motivos me fizeram a esta escolha:
	
			1. A quantidade de comentários em uma atividade é algo difícil de ser mensurado antecipadamente, o que pode levar a coleção crescer além da
				"folga" determinada. Isso levaria a um cenário de realocações de documentos e posterior fragmentação da coleção.
			2. Em cenários bastante específico é que o usuario final deseja ver os comentários. Não faz sentido sempre que buscar uma atividade já trazer todos
				os seus comentários.Inclusive, essa busca pode ( e deve ) ser feito com buscas parciais ( usando limit e skip ).
		
	activity-comments: [{
	    activity_id: objectId
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
					 activities: [
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
					 activities: [
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
			tags:["governo","clima","desmatamento"],
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
					 activities: [
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
					 activities: [
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
	
	//Retorna 5 IDs de usuários consecutivos a partir de uma posição especificada
	function getFiveUserIDs ( fromIndex ) {
	    var vetUserID = [];
		if ( fromIndex >= 0 && fromIndex <= 5 ) {
		  for (var i=fromIndex;i<fromIndex+5;i++) {
			vetUserID.push(usu_array[i]._id);
		  }
		}
		return vetUserID;
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
		
		var vetUserIDs = getFiveUserIDs (j);
		
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
			if (goal.activities) {
				goal.activities.forEach(function(activity){
					activity.startDate = getRandomDateBetween(goal.startDate,goal.endDate);
					activity.endDate = getRandomDateBetween(activity.startDate,goal.endDate);
					activity.dreamDate = getRandomDateBetween(activity.startDate,activity.endDate);
					activity.realocate = getRandomBoolean();
					activity.expired = getRandomBoolean();
					activity.tags =  [];
					activity.historics = null;	
					activity.members = [];
				});
			}
		});
		db.projects.insert(project);	
	}
```	
> db.projects.count()
```
	5
```
> db.projects.findOne()
```
	{
        "_id" : ObjectId("569cd21a5114717dd2af2a9d"),
        "name" : "Carro movido à água",
        "description" : "Construção de automóveis que utilizam água como combustível.",
        "startDate" : ISODate("2016-09-05T18:06:09.140Z"),
        "endDate" : ISODate("2016-12-15T22:49:56.078Z"),
        "dreamDate" : ISODate("2016-12-02T05:13:31.007Z"),
        "visible" : true,
        "realocate" : false,
        "expired" : true,
        "visualizableMod" : false,
        "members" : [
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a93"),
                        "type" : "idealizador",
                        "notify" : true
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a94"),
                        "type" : "engenheiro",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a95"),
                        "type" : "mecanico",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a96"),
                        "type" : "eletricista",
                        "notify" : false
                },
                {
                        "user_id" : ObjectId("569cd0535114717dd2af2a97"),
                        "type" : "montador",
                        "notify" : false
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
                        "startDate" : ISODate("2016-11-17T00:24:42.534Z"),
                        "endDate" : ISODate("2016-12-02T17:42:52.961Z"),
                        "dreamDate" : ISODate("2016-11-18T00:30:41.013Z"),
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
                                        "name" : "Montagem do circuito básico",
                                        "description" : "Montar o circuito básico já trará grandes avanços nas evoluções da idéias.",
                                        "startDate" : ISODate("2016-11-28T06:55:49.577Z"),
                                        "endDate" : ISODate("2016-11-30T17:59:57.839Z"),
                                        "dreamDate" : ISODate("2016-11-28T10:23:48.904Z"),
                                        "realocate" : true,
                                        "expired" : true,
                                        "tags" : [ ],
                                        "historics" : null,
                                        "members" : [ ]
                                },
                                {
                                        "name" : "Construção do protótipo",
                                        "description" : "Confeccionar um modelo miniatura do veículo.",
                                        "startDate" : ISODate("2016-11-17T01:14:27.058Z"),
                                        "endDate" : ISODate("2016-11-24T01:16:47.980Z"),
                                        "dreamDate" : ISODate("2016-11-19T02:20:52.771Z"),
                                        "realocate" : true,
                                        "expired" : true,
                                        "tags" : [ ],
                                        "historics" : null,
                                        "members" : [ ]
                                }
                        ]
                }
        ]
	}
```
```

	> db.projects.find({}).forEach(function(project){ 
		project.members.forEach(function(member){ 
		   print("project: " + project._id + " -> member: " + member.user_id ); 
	   }); 
	  }) 
  
	project: 569cd21a5114717dd2af2a9d -> member: 569cd0535114717dd2af2a93
	project: 569cd21a5114717dd2af2a9d -> member: 569cd0535114717dd2af2a94
	project: 569cd21a5114717dd2af2a9d -> member: 569cd0535114717dd2af2a95
	project: 569cd21a5114717dd2af2a9d -> member: 569cd0535114717dd2af2a96
	project: 569cd21a5114717dd2af2a9d -> member: 569cd0535114717dd2af2a97
	project: 569cd21a5114717dd2af2a9e -> member: 569cd0535114717dd2af2a94
	project: 569cd21a5114717dd2af2a9e -> member: 569cd0535114717dd2af2a95
	project: 569cd21a5114717dd2af2a9e -> member: 569cd0535114717dd2af2a96
	project: 569cd21a5114717dd2af2a9e -> member: 569cd0535114717dd2af2a97
	project: 569cd21a5114717dd2af2a9e -> member: 569cd0535114717dd2af2a98
	project: 569cd21a5114717dd2af2a9f -> member: 569cd0535114717dd2af2a95
	project: 569cd21a5114717dd2af2a9f -> member: 569cd0535114717dd2af2a96
	project: 569cd21a5114717dd2af2a9f -> member: 569cd0535114717dd2af2a97
	project: 569cd21a5114717dd2af2a9f -> member: 569cd0535114717dd2af2a98
	project: 569cd21a5114717dd2af2a9f -> member: 569cd0535114717dd2af2a99
	project: 569cd21a5114717dd2af2aa0 -> member: 569cd0535114717dd2af2a96
	project: 569cd21a5114717dd2af2aa0 -> member: 569cd0535114717dd2af2a97
	project: 569cd21a5114717dd2af2aa0 -> member: 569cd0535114717dd2af2a98
	project: 569cd21a5114717dd2af2aa0 -> member: 569cd0535114717dd2af2a99
	project: 569cd21a5114717dd2af2aa0 -> member: 569cd0535114717dd2af2a9a
	project: 569cd21a5114717dd2af2aa1 -> member: 569cd0535114717dd2af2a97
	project: 569cd21a5114717dd2af2aa1 -> member: 569cd0535114717dd2af2a98
	project: 569cd21a5114717dd2af2aa1 -> member: 569cd0535114717dd2af2a99
	project: 569cd21a5114717dd2af2aa1 -> member: 569cd0535114717dd2af2a9a
	project: 569cd21a5114717dd2af2aa1 -> member: 569cd0535114717dd2af2a9b

```

## Teste Bold
