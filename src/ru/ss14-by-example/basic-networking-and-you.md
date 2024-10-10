Вот профессиональный перевод для mdBook:

# Основы сетевого взаимодействия и вы

Вы уже должны быть знакомы с парадигмой `Клиент`/`Сервер`, используемой в Robust. Если нет, ознакомьтесь с предыдущей документацией.

SS14 — это многопользовательская игра! Это очень важно, и этот факт нужно учитывать при программировании, чтобы избежать ошибок и проблем с безопасностью.

Чаще всего сетевая работа заключается в том, что сервер отправляет важные данные клиенту, чтобы клиент мог использовать их, например, для отображения интерфейса пользователя или визуальных эффектов. Это называется **репликацией**, и в SS14 она в основном реализуется через **состояния компонентов**.

## Состояния компонентов

Состояние компонента — это простая структура данных, наследуемая от `ComponentState`. Она определяет, какие данные отправляются клиенту, и собирается из данных внутри компонентов.

Как игра узнаёт, когда нужно отправить данные? Очевидно, что данные не отправляются постоянно — это было бы чрезмерно и плохо сказывалось бы на производительности и пропускной способности. Вместо этого сервер должен вызвать `Dirty(EntityUid uid, Component component)`, чтобы пометить сущность как "грязную", что означает создание и отправку нового состояния компонента в следующий тик.

Для отправки и получения данных через состояния компонентов существуют два специальных события: `ComponentGetState` и `ComponentHandleState`. `GetState` всегда вызывается на сервере, а `HandleState` — на клиенте. Однако подписки на оба события можно разместить в разделе `Shared`, и это будет работать должным образом!

### Автоматическая генерация состояний компонентов

Robust Toolbox поддерживает использование **генераторов исходного кода** для значительного упрощения сетевого взаимодействия состояний компонентов. Это предпочтительный способ в большинстве случаев, так как он позволяет избежать ручного написания однотипного кода. Это реализуется с помощью функции C#, которая анализирует код до его компиляции и автоматически генерирует необходимые фрагменты кода.

Во-первых, ваш компонент и все сетевые поля должны находиться в разделе `Content.Shared`, а класс компонента должен быть помечен атрибутом `[NetworkedComponent]`, который активирует сетевое взаимодействие.

Чтобы использовать генератор исходного кода для автоматической репликации полей, сделайте класс компонента частичным, пометьте его атрибутом `[AutoGenerateComponentState]` и укажите поля, которые нужно реплицировать, с помощью `[AutoNetworkedField]`. Затем, когда вы помечаете компонент как грязный (или когда он впервые добавляется к сущности), всё должно работать автоматически, и клиент получит все сетевые поля.

Если у вас есть код, который должен выполняться после обновления полей (например, обновление внешнего вида), измените атрибут состояния компонента на `[AutoGenerateComponentState(true)]`, и тогда вы сможете подписаться на событие AfterAutoHandleStateEvent и выполнить необходимые действия там.

Если ваше поле требует клонирования для целей предсказания (например, словарь), вы можете изменить атрибут поля на `[AutoNetworkedField(true)]`. Если требуется более сложное сетевое взаимодействие, следует использовать ручной метод.

Пример всего необходимого сетевого кода для компонента ID-карты можно найти здесь: https://github.com/space-wizards/space-station-14/pull/14845:
```csharp
// IDCardComponent.cs
[RegisterComponent, NetworkedComponent]
[AutoGenerateComponentState]
public sealed partial class IdCardComponent : Component
{
    [DataField]
    [AutoNetworkedField]
    public string? FullName;

    [DataField]
    [AutoNetworkedField]
    public string? JobTitle;
}
```

### Ручная обработка состояния компонентов

Иногда приходится использовать ручной метод, если обработка данных сложнее, чем просто установка полей.

Рассмотрим пример с компонентом фоновых звуков (хотя в данном случае это можно было бы сгенерировать автоматически):

```csharp
// AmbientSoundComponent.cs
    [Serializable, NetSerializable]
    public sealed class AmbientSoundComponentState : ComponentState
    {
        public bool Enabled { get; init; }
        public float Range { get; init; }
        public float Volume { get; init; }
    }
```

Это определение простого состояния компонента. Оно помечено атрибутами `[Serializable, NetSerializable]`, что необходимо для любого объекта, отправляемого по сети. Этот класс определяет три переменные, которые необходимо синхронизировать с клиентом: включён ли звук, его радиус и громкость.

Теперь посмотрим, как это состояние создается на сервере:

```csharp
/// SharedAmbientSoundSystem.cs
        public override void Initialize()
        {
            base.Initialize();
            SubscribeLocalEvent<AmbientSoundComponent, ComponentGetState>(GetCompState);
            SubscribeLocalEvent<AmbientSoundComponent, ComponentHandleState>(HandleCompState);
        }

        ...

// В обработчиках событий...
        private void GetCompState(Entity<AmbientSoundComponent> ent, ref ComponentGetState args)
        {
            args.State = new AmbientSoundComponentState
            {
                Enabled = ent.Comp.Enabled,
                Range = ent.Comp.Range,
                Volume = ent.Comp.Volume,
            };
        }
```

Важно отметить, что в аргументах обработчика событий используется синтаксис `ref ComponentGetState args` вместо простого `ComponentGetState args`. Это необходимо для некоторых событий, так как они являются структурами, передаваемыми "по ссылке", а не классами, наследуемыми от `EntityEventArgs`. Это сделано для повышения производительности и может быть важным для предотвращения ошибок во время выполнения, если забыть про `ref`.

Чтобы указать состояние для отправки, достаточно установить поле `State` в событии, используя новое состояние компонента, созданное на основе значений с сервера. Просто!

---

Теперь рассмотрим клиентскую сторону (хотя технически это всё ещё общий код, но он выполняется только на клиенте!):

```csharp
/// SharedAmbientSoundSystem.cs
        ...

        private void HandleCompState(Entity<AmbientSoundComponent> ent, ref ComponentHandleState args)
        {
            if (args.Current is not AmbientSoundComponentState state)
                return;

            ent.Comp.Enabled = state.Enabled;
            ent.Comp.Range = state.Range;
            ent.Comp.Volume = state.Volume;
        }
```

Здесь снова используется `ref`.

Первая строка метода выполняет проверку с помощью сопоставления с шаблоном C# для приведения поля `Current` в событии к ожидаемому состоянию. После этого клиент использует данные из состояния компонента для синхронизации своего компонента, чтобы любая клиентская специфическая логика (например, воспроизведение звуков) работала так, как это задумано сервером.

### Пример сетевого взаимодействия компонентов

В качестве примера высокого уровня рассмотрим, как атмосферные венты управляют своими фоновыми звуками.

```csharp
/// GasVentPumpSystem.cs
            ...
            _ambientSoundSystem.SetAmbience(uid, true);
            if (!vent.Enabled)
            {
                _ambientSoundSystem.SetAmbience(uid, false);
            ...
```

Вентиль сначала включает фоновый звук по умолчанию. Однако если вентиль отключён, он выключает звук.

Всё это выполняется в серверном коде! В функции `SetAmbience` система звуков вызывает `Dirty` для сущности, что сообщает серверу об обновлении данных сущности, и клиент должен об этом узнать. Затем в следующий тик сервер поднимает событие `ComponentGetState` для вентиля, и состояние фонового звука создаётся и отправляется.

Когда клиент его получает (с учётом задержки), он вызывает событие `ComponentHandleState` для вентиля, что приводит к отключению звука на клиенте. Удобно!

## Сетевые события

Другим основным способом связи между сервером и клиентом, помимо репликации через состояния компонентов, являются **сетевые события** и более низкоуровневый механизм `NetMessage`.

Сетевые события противопоставляются локальным событиям (`RaiseLocalEvent` или `SubscribeLocalEvent`), которые происходят исключительно на одной стороне сети. В то время как сетевые события передаются исключительно по сети с использованием `RaiseNetworkEvent` и `SubscribeNetworkEvent`.

Сетевые события могут содержать произвольные данные, не связанные с каким-либо компонентом или сущностью, и могут быть отправлены в любой момент, как с клиента, так и с сервера. **Когда сервер обрабатывает сетевое событие, отправленное с клиента, следует проявлять осторожность и относиться к данным как к недостоверным, поскольку злоумышленники могут отправлять любые данные.**

`NetMessage` — это низкоуровневый эквивалент сетевых событий (фактически, сетевые события просто создают `NetMessage`). Избегайте его использования, если вы точно не знаете, что делаете, поэтому здесь я не буду его детально рассматривать.

### Пример

Рассмотрим пример с админхелпом (или системой *bwoink*), который отправляет произвольные данные клиентам.

Вот как определено сетевое событие:

```csharp
/// SharedBwoinkSystem.cs

...
    
        [Serializable, NetSerializable]
        public sealed class BwoinkTextMessage : EntityEventArgs




Вот продолжение перевода:

---

```csharp
/// SharedBwoinkSystem.cs

...
    
        [Serializable, NetSerializable]
        public sealed class BwoinkTextMessage : EntityEventArgs
        {
            public DateTime SentAt { get; }
            public NetUserId ChannelId { get; }
            public NetUserId TrueSender { get; }
            public string Text { get; }

            public BwoinkTextMessage(NetUserId channelId, NetUserId trueSender, string text, DateTime? sentAt = default)
            {
                SentAt = sentAt ?? DateTime.Now;
                ChannelId = channelId;
                TrueSender = trueSender;
                Text = text;
            }
        }
```

Обратите внимание, что это практически идентично обычному классу данных событий, за исключением того, что он помечен как `NetSerializable`, по тем же причинам, которые упоминались выше для состояний компонентов. Я не буду подробно останавливаться на конкретных данных здесь — думаю, вы понимаете суть.

Интересная особенность этого события заключается в том, что оно поднимается и обрабатывается как на клиенте, так и на сервере. Так что давайте рассмотрим их по отдельности.

#### От клиента к серверу

```csharp
/// Content.Client ... BwoinkSystem.cs
...
        public void Send(NetUserId channelId, string text)
        {
            // Используем идентификатор канала как "настоящего отправителя".
            // Сервер проигнорирует это, и если кто-то заставит его не игнорировать (что плохо, позволяет подделывать сообщения!!!), это поможет.
            RaiseNetworkEvent(new BwoinkTextMessage(channelId, channelId, text));
        }
```

`Send` вызывается всякий раз, когда в интерфейс BWOINK (tm) вводится текст:

```csharp
/// BwoinkPanel.xaml.cs
...
        private void Input_OnTextEntered(LineEdit.LineEditEventArgs args)
        {
            if (string.IsNullOrWhiteSpace(args.Text))
                return;

            _bwoinkSystem.Send(ChannelId, args.Text);
            SenderLineEdit.Clear();
        }
...
```

Все просто! Клиент вводит сообщение, нажимает Enter, и система Bwoink создает сетевое событие из сообщения и отправляет его на сервер. Давайте посмотрим, как оно обрабатывается на сервере:

#### Обработка на сервере

Обработчик этого события довольно объемный, поэтому я сокращу его до важного:

```csharp
/// Content.Server ... BwoinkSystem.cs

        // это фактически в общем коде и переопределено на сервере/клиенте, но вы понимаете идею для простоты...
        public override void Initialize()
        {
            base.Initialize();

            SubscribeNetworkEvent<BwoinkTextMessage>(OnBwoinkTextMessage);
        }

        ...

        protected override void OnBwoinkTextMessage(BwoinkTextMessage message, EntitySessionEventArgs eventArgs)
        {
            base.OnBwoinkTextMessage(message, eventArgs);
            var senderSession = (IPlayerSession) eventArgs.SenderSession;

            // TODO: Санитизация текста?
            // Убедитесь, что это лицо действительно имеет право отправлять сообщение здесь.
            var personalChannel = senderSession.UserId == message.ChannelId;
            var senderAdmin = _adminManager.GetAdminData(senderSession);
            var authorized = personalChannel || senderAdmin != null;
            if (!authorized)
            {
                // Неавторизованный bwoink (лог?)
                return;
            }

            var escapedText = FormattedMessage.EscapeText(message.Text);

            var bwoinkText = ...

            var msg = new BwoinkTextMessage(message.ChannelId, senderSession.UserId, bwoinkText);

            ...
            
            // Админы
            var targets = _adminManager.ActiveAdmins.Select(p => p.ConnectedClient).ToList();

            // И вовлеченный игрок
            if (_playerManager.TryGetSessionById(message.ChannelId, out var session))
                if (!targets.Contains(session.ConnectedClient))
                    targets.Add(session.ConnectedClient);

            foreach (var channel in targets)
                RaiseNetworkEvent(msg, channel);
            
            ...
```

Первое, что делает этот обработчик — проверяет, действительно ли клиент имеет право отправить это сообщение! Он проверяет, активен ли пользователь в текущем тикете или является ли он администратором. Просто. **Всегда проверяйте события, отправленные клиентом!**

Затем сообщение пересылается различным сторонам, чтобы они могли его увидеть. Сначала собираются все активные администраторы, затем добавляется игрок, участвующий в тикете, и сообщение отправляется обратно.

#### Обработка на клиенте

Теперь вернемся к клиенту, чтобы посмотреть, что он делает с новым текстовым сообщением, отправленным с сервера.

```csharp
/// Content.Client ... BwoinkSystem.cs
        // это фактически в общем коде и переопределено на сервере/клиенте, но вы понимаете идею для простоты...
        public override void Initialize()
        {
            base.Initialize();

            SubscribeNetworkEvent<BwoinkTextMessage>(OnBwoinkTextMessage);
        }

        ...

        protected override void OnBwoinkTextMessage(BwoinkTextMessage message, EntitySessionEventArgs eventArgs)
        {
            base.OnBwoinkTextMessage(message, eventArgs);
            LogBwoink(message);
            // Визуализация сообщения
            var window = EnsurePanel(message.ChannelId);
            window.ReceiveLine(message);
            // Проигрываем звук, если это сообщение не отправил текущий пользователь
            var localPlayer = _playerManager.LocalPlayer;
            if (localPlayer?.UserId != message.TrueSender)
            {
                SoundSystem.Play(Filter.Local(), "/Audio/Effects/adminhelp.ogg");
                _clyde.RequestWindowAttention();
            }

            _adminWindow?.OnBwoink(message.ChannelId);
        }
```

Этот код довольно прост: он открывает окно ahelp, отправляет строку сообщения в окно для визуализации, а затем воспроизводит **Забавный звук** (если сообщение не было отправлено самим пользователем). Проверки здесь почти не нужны, потому что сервер может доверять данным, отправленным клиенту.

---

## Potentially Visible Set (PVS)

Вы, вероятно, уже слышали термин "Potentially Visible Set" (PVS), если интересовались разработкой, так как это важная часть системы.

Подумайте на секунду: в игре очень много сущностей! И, скорее всего, многие из них постоянно вызывают `Dirty`, а клиентов тоже будет много. Как мы можем определить, какие состояния отправлять каждому клиенту? Ответ заключается в **системе Potentially Visible Set (PVS)**.

Это не будет детальным объяснением на низком уровне, но по сути, **PVS** основан на "чанках" и отправляет состояния компонентов только тем клиентам, которые находятся в пределах видимости сущности.