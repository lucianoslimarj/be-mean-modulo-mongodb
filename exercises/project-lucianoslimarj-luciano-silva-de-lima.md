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
			beginDate: Date,
			dreamDate: Date,
			endDate: Date,
			realocate: Boolean,
			expired: Boolean
			tags: [String],
			historics: [{realocateDate: Date}]
			activities: [{
				name: String,
				description: String,
				beginDate: Date,
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
```
	## 1. Cadastre 10 usuários diferentes

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
			var millisInOneYear = 365*24*60*60*1000;
			var usu = {
				name: usuario,
				bio:  ((posBlank>=0)?usuario.substring(0,posBlank):usuario) + " bio",
				//registerDate = data atual subtraido de até um ano.
				registerDate: new Date((new Date()).getTime() - ( _rand() * millisInOneYear)),
				avatarPath: "http://s3.amazonaws.com/avatar/" + usuario.replace(/\s/g,'') + ".jpeg",
				settings: { backgroundPath:"http://s3.amazonaws.com/background/" + usuario.replace(/\s/g,'') + "/"},
				auth: {
					username:  usuario.toLowerCase().replace(/\s/g,''),
					email: usuario.toLowerCase().replace(/\s/g,'.') + "@gmail.com",
					password:"12345",
					lastAccess:null,
					online: (_rand()* 10) > 5,
					disabled: (_rand()* 10) > 5,
					hashToken: "AaYYYyyNOPGaa_"
				},
			};
			//lastAccess = data do registro menos um quantidade de dias aleatória entre 0 e a diferença de dias entre a data corrente e a data de registro.
			usu.auth.lastAccess = new Date(usu.registerDate.getTime() +  ( _rand() * ((new Date()).getTime() - usu.registerDate.getTime()) ));
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
				"_id" : ObjectId("56976f3db2e3492ffc79d4f8"),
				"name" : "Francisco Antunes",
				"bio" : "Francisco bio",
				"registerDate" : ISODate("2015-02-24T01:03:53.410Z"),
				"avatarPath" : "http://s3.amazonaws.com/avatar/FranciscoAntunes.jpeg",
				"settings" : {
						"backgroundPath" : "http://s3.amazonaws.com/background/FranciscoAntunes/"
				},
				"auth" : {
						"username" : "franciscoantunes",
						"email" : "francisco.antunes@gmail.com",
						"password" : "12345",
						"lastAccess" : ISODate("2015-10-25T23:04:52.491Z"),
						"online" : true,
						"disabled" : false,
						"hashToken" : "AaYYYyyNOPGaa_"
				}
		}
```		
> db.users.find({},{"_id":1});
```		
		{ "_id" : ObjectId("56976f3db2e3492ffc79d4f8") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4f9") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4fa") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4fb") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4fc") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4fd") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4fe") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d4ff") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d500") }
		{ "_id" : ObjectId("56976f3eb2e3492ffc79d501") }
```		
  
## Retrieve - busca

## Update - alteração

## Delete - remoção

## Sharding
// coloque aqui todos os comandos que você executou

## Replica
// coloque aqui todos os comandos que você executou

0019 114,68
7064  100,80