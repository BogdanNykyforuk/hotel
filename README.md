# hotel
Документація до системи бронювання готелів
Опис системи
Ця система бронювання готелів реалізована на Python з використанням різних патернів проектування. Вона забезпечує можливість створення та управління бронюваннями, управління готелями та номерами, а також інформування користувачів про статус їхніх бронювань.

Основні компоненти системи
База даних (Singleton)
Фабрика бронювань (Factory Method)
Система сповіщень (Observer)
Стратегії бронювання (Strategy)
Команди (Command)
Інтерактивний інтерфейс (CLI)
База даних (Singleton)
База даних використовується для зберігання інформації про готелі, номери, користувачів та бронювання.


class Database:
    # Singleton для бази даних
    _instance = None

    def __new__(cls, *args, **kwargs):
        if not cls._instance:
            cls._instance = super(Database, cls).__new__(cls, *args, **kwargs)
        return cls._instance

    def __init__(self):
        self.hotels = {}
        self.bookings = {}
        self.users = {}

    def add_hotel(self, hotel):
        self.hotels[hotel.id] = hotel

    def get_hotel(self, hotel_id):
        return self.hotels.get(hotel_id)

    def add_booking(self, booking):
        self.bookings[booking.id] = booking

    def get_booking(self, booking_id):
        return self.bookings.get(booking_id)

    def add_user(self, user):
        self.users[user.id] = user

    def get_user(self, user_id):
        return self.users.get(user_id)
Фабрика бронювань (Factory Method)
Цей патерн використовується для створення об'єктів бронювань.


class Booking(ABC):
    def __init__(self, user_id, hotel_id, room_id):
        self.id = uuid.uuid4()
        self.user_id = user_id
        self.hotel_id = hotel_id
        self.room_id = room_id
        self.status = "Pending"

    @abstractmethod
    def book(self):
        pass

class HotelBooking(Booking):
    def book(self):
        self.status = "Booked"
        return f"Hotel booking {self.id} confirmed!"

class BookingFactory(ABC):
    @abstractmethod
    def create_booking(self, user_id, hotel_id, room_id):
        pass

class HotelBookingFactory(BookingFactory):
    def create_booking(self, user_id, hotel_id, room_id):
        return HotelBooking(user_id, hotel_id, room_id)
Система сповіщень (Observer)
Цей патерн використовується для інформування користувачів про статус їхніх бронювань.


class Observer(ABC):
    @abstractmethod
    def update(self, message: str):
        pass

class User(Observer):
    def __init__(self, user_id):
        self.id = user_id

    def update(self, message: str):
        print(f"User {self.id} received: {message}")

class BookingSystem:
    def __init__(self):
        self._observers = []

    def add_observer(self, observer: Observer):
        self._observers.append(observer)

    def remove_observer(self, observer: Observer):
        self._observers.remove(observer)

    def notify_observers(self, message: str):
        for observer in self._observers:
            observer.update(message)

    def create_booking(self, booking):
        db.add_booking(booking)
        self.notify_observers(f"Booking {booking.id} has been created with status {booking.status}")
Стратегії бронювання (Strategy)
Цей патерн дозволяє використовувати різні алгоритми бронювання.


class BookingStrategy(ABC):
    @abstractmethod
    def book(self, booking):
        pass

class CheapestStrategy(BookingStrategy):
    def book(self, booking):
        booking.status = "Booked (Cheapest)"
        return f"Booked using the cheapest strategy: {booking.id}"

class FastestStrategy(BookingStrategy):
    def book(self, booking):
        booking.status = "Booked (Fastest)"
        return f"Booked using the fastest strategy: {booking.id}"

class BookingContext:
    def __init__(self, strategy: BookingStrategy):
        self._strategy = strategy

    def set_strategy(self, strategy: BookingStrategy):
        self._strategy = strategy

    def book(self, booking):
        return self._strategy.book(booking)
Команди (Command)
Цей патерн дозволяє інкапсулювати запити на бронювання та їх виконання.


class Command(ABC):
    @abstractmethod
    def execute(self):
        pass

class BookCommand(Command):
    def __init__(self, booking):
        self._booking = booking

    def execute(self):
        return self._booking.book()

class CancelCommand(Command):
    def __init__(self, booking_id):
        self._booking_id = booking_id

    def execute(self):
        booking = db.get_booking(self._booking_id)
        if booking:
            booking.status = "Cancelled"
            return f"Booking {self._booking_id} cancelled"
        return "Booking not found"

class BookingInvoker:
    def __init__(self):
        self._commands = []

    def add_command(self, command: Command):
        self._commands.append(command)

    def execute_commands(self):
        results = []
        for command in self._commands:
            results.append(command.execute())
        self._commands.clear()
        return results
Інтерактивний інтерфейс (CLI)
Інтерфейс командного рядка дозволяє користувачам взаємодіяти з системою.


import cmd

class BookingCLI(cmd.Cmd):
    intro = 'Welcome to the hotel booking system. Type help or ? to list commands.\n'
    prompt = '(booking) '
    booking_system = BookingSystem()

    def do_add_hotel(self, arg):
        'Add a new hotel: add_hotel hotel_id hotel_name'
        args = arg.split()
        if len(args) < 2:
            print("Usage: add_hotel hotel_id hotel_name")
            return
        hotel_id, hotel_name = args[0], ' '.join(args[1:])
        db.add_hotel(Hotel(hotel_id, hotel_name, []))
        print(f"Hotel {hotel_name} added with id {hotel_id}")

    def do_add_room(self, arg):
        'Add a room to a hotel: add_room hotel_id room_id room_type'
        args = arg.split()
        if len(args) < 3:
            print("Usage: add_room hotel_id room_id room_type")
            return
        hotel_id, room_id, room_type = args[0], args[1], ' '.join(args[2:])
        hotel = db.get_hotel(hotel_id)
        if not hotel:
            print(f"Hotel with id {hotel_id} not found")
            return
        hotel.rooms.append(Room(room_id, room_type))
        print(f"Room {room_id} of type {room_type} added to hotel {hotel_id}")

    def do_add_user(self, arg):
        'Add a new user: add_user user_id'
        user_id = arg.strip()
        if not user_id:
            print("Usage: add_user user_id")
            return
        db.add_user(User(user_id))
        print(f"User {user_id} added")

    def do_create_booking(self, arg):
        'Create a booking: create_booking user_id hotel_id room_id'
        args = arg.split()
        if len(args) < 3:
            print("Usage: create_booking user_id hotel_id room_id")
            return
        user_id, hotel_id, room_id = args[0], args[1], args[2]
        booking_factory = HotelBookingFactory()
        new_booking = booking_factory.create_booking(user_id, hotel_id, room_id)
        self.booking_system.add_observer(db.get_user(user_id))
        self.booking_system.create_booking(new_booking)
        print(new_booking.book())
        self.booking_system.notify_observers(f"Booking {new_booking.id} status updated to {new_booking.status}")

    def do_cancel_booking(self, arg):
        'Cancel a booking: cancel_booking booking_id'
        booking_id = arg.strip()
        if not booking_id:
            print("Usage: cancel_booking booking_id")
            return
        cancel_command = CancelCommand(booking_id)
        invoker = BookingInvoker()
        invoker.add_command(cancel_command)
        print(invoker.execute_commands()[0])

    def do_set_strategy(self, arg):
        'Set booking strategy: set_strategy [cheapest|fastest]'
        if arg not in ['cheapest', 'fastest']:
            print("Usage: set_strategy [cheapest|fastest]")
            return
        strategy = CheapestStrategy() if arg == 'cheapest' else FastestStrategy()
        booking_context.set_strategy(strategy)
        print(f"Strategy set to {arg}")

    def do_quit(self, arg):
        'Quit the application'
        print('Thank you for using the booking system.')
        return True

if __name__ == '__main__':
    booking_context = BookingContext(CheapestStrategy())
    BookingCLI().cmdloop()
Інструкція з використання
Запустіть скрипт.
Використовуйте команди для взаємодії з системою.
Приклади команд
Додавання готелю:


add_hotel hotel1 "Hotel Sunshine"
Додавання номера в готель:


add_room hotel1 room1 "Single"
Додавання користувача:

add_user user1
Створення бронювання:


create_booking user1 hotel1 room1
Скасування бронювання:


cancel_booking <booking_id>
Встановлення стратегії бронювання:


set_strategy cheapest
set_strategy fastest
Вихід із програми:


quit
