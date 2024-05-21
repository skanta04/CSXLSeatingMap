# CSXL Seating Map Feature Technical Specification

This document contains the technical specifications, including sample data representation of our feature, descriptions of underlying database / entity-level representation decisions and development concerns.

# Authors

- [Saishreeya Kantamsetty](https://github.com/skanta04)
- [Upasana Lamsal](https://github.com/upasanal)
- [Ellie Kim](https://github.com/ellieekimm)
- [Jake Rogers](https://github.com/Jakerogers1)

# Table of Contents

- [Introduction to Demo](#introduction-to-demo)
  - [How do I reserve a seat?](#how-do-i-reserve-a-seat)
  - [How do I view my seat reservation?](#how-do-i-view-my-room-reservation)
  - [How do I cancel my seat reservation](#how-do-i-cancel-my-room-reservation)
- [Description and Sample Data](#descriptions-and-sample-data-representation-of-feature)
  - [0. Room Partial](#0-room-partial)
  - [1. Reservation Request](#1-reservation-request)
  - [2. Reservation Service](#2-reservation-service)
  - [3. Route to get all upcoming reservations for a user.](#3-route-to-get-all-upcoming-reservations-for-a-user)
- [Underlying Database / Entity-Level Representation decisions](#underlying-database--entity-level-representation-decisions)
- [Technical and User Experience Design Choices](#technical-and-user-experience-design-choices)
- [Development Concerns](#development-concerns)
- [For Future Developers](#future-developers)


# Introduction to Demo

This section of the documents contains the instructions to replicate the steps we take in our demo. By reading this, you should be able to replicate how we create seat reservations, view upcoming seat reservations, and delete seat reservations. We would recommend running the `reset_demo` script to populate the backend with some sample reservations to ease the testing process.

## How do I reserve a seat?

- From the sidebar, click on Coworking > Reserve Seats
- Now you should see a seat map, with selectable seats. To reserve a seat click on the desired seat on the map. You can select up to 4 seats, each reserved for a 2-hour time period. Selecting more than 4 is not allowed as the fifth seat attempted to select will not be selected. The seating limits and periods that seats are reserved for are set by a function in the frontend and backend service files and can be adjusted if needed.
  -While selecting seats, you will notice a legend with 4 colors. The structure is as follows:
  - <span style="color: #03691e"> Seating Monitors (Green)</span>: These seats represent seating monitors seats that can be found within the floorplan. Once selected the seat will turn red, and once reserved this seat will turn grey.
  - <span style="color: #4d4d4d"> Unavailable Seats (Gray)</span>: These slots are unable to be reserved because they are currently reserved by another user.
  - <span style="color: #3479be"> Communal Area Seats (Blue)</span>: These seats represent your communual area seats. Similar to the seating monitors, these seats will turn red upon selection and be greyed out upon being reserved
  - <span style="color: #REPLACEME"> Standing Monitor Seats (Yellow)</span>: These seats represent standing monitors seats that can be found within the floorplan. Once selected the seat will turn red, and once reserved this seat will turn grey.
 
<img width="904" alt="image" src="https://github.com/comp423-24s/csxl-final-team-a7/assets/115567471/f64a83dd-9c5e-453f-af2d-83e800d81dc6">


## Make Reservations

- Once you have picked the seats you like, click on the **Reserve Seats** button, which will draft a reservation for you, and redirect you to the confirmation page.
- On the confirmation page, you can view the details of your selection including the time, location, seats, and date for your reservation. If you are happy with your selection, click on the **Confirm** button. Otherwise, click on **Cancel**. Note that your reservation draft will automatically be cancelled within 5 minutes if you don't press anything. Navigating out of the page also cancels your reservation.

<img width="904" alt="image" src="https://github.com/comp423-24s/csxl-final-team-a7/assets/115567471/054dac50-02e9-4b3d-a431-2e695553a836">


## View Reservations

- From the sidebar, click on Coworking.

- Now you should be able to view your upcoming reservation below the open card and as well as under the "Upcoming Reservations" header.

<img width="904" alt="image" src="https://github.com/comp423-24s/csxl-final-team-a7/assets/115567471/5ff23b7e-f66f-44e1-a02f-6f6d95955e65">


## How do I cancel my reservation?

- From the sidebar, click on Coworking
- On the card for the reservation you want to cancel, simply click on the cancel button. Note that this feature doesn't exist for active reservations, since you are already checked-in. You can instead simply check-out for active reservations.

## Where can I find my active reservations?

- Once you have checked in, the reservation becomes active. The active reservations can be found on the Coworking page.

# Descriptions and Sample Data Representation of feature

We have added / modified the following models / API routes:

## 0. Room Service

The room service plays a crucial role in facilitating the UI of the seating map by providing methods to retrieve floorplan details give a room ID, including table layouts and boundaries. We have added the following room service methods into the under `backend/services/room.py`.  

```py3
     def get_seats_by_id(self, id: str) -> Optional[list[SeatDetails]]:
        room_entity = self._session.query(RoomEntity).filter_by(id=id).first()
        if room_entity is None: 
            raise ResourceNotFoundException(f"Room with id: {id} does not exist.")
        seats = [seat_entity.to_model() for seat_entity in room_entity.seats]
        return seats

    def get_floorplan_by_room_id(self, id: str) -> Optional[FloorplanDetails]:
        """Get the floorplan with the given id of the room."""
        room_entity = self._session.query(RoomEntity).filter_by(id=id).first()
        if room_entity is None: 
            raise ResourceNotFoundException(f"Room with id: {id} does not exist.")
        if room_entity.floorplan is not None: 
            return room_entity.floorplan.to_model()

    def get_boundaries_by_room_id(self, id: str) -> Optional[str]:
        floorplan_model = self.get_floorplan_by_room_id(id)
        if floorplan_model is not None: 
            return floorplan_model.boundaries
        else: 
            return None

    def get_circletables_by_room_id(self, id: str) -> Optional[list[CircleTableDetails]]:
        floorplan_model = self.get_floorplan_by_room_id(id)
        if floorplan_model is not None: 
            return floorplan_model.circle_tables
        else: 
            return None

    def get_rectangletables_by_room_id(self, id: str) -> Optional[list[RectangleTableDetails]]:
        floorplan_model = self.get_floorplan_by_room_id(id)
        if floorplan_model is not None: 
            return floorplan_model.rectangle_tables
        else: 
            return None
```

## 1. Room API

The Room API exposes endpoints for obtaining floorplan details necessary for administrative reservation management. These endpoints ensure users can view available seating arrangements without accessing user sensitive information. This can be found under `api/room.py`.

```py3
@api.get("/{id}/seats", response_model=list[SeatDetails], tags=["Room"])
def get_seats_by_room(
    id: str,
    room_service: RoomService = Depends(),
) -> Optional[list[SeatDetails]]:
    """Get seats by room
    Parameters:
        id: a string representing a unique identifier for room
        room_service: a valid roomService
    """
    return room_service.get_seats_by_id(id)


@api.get("/{id}/floorplan", response_model=FloorplanDetails, tags=["Rooms"])
def get_floorplan(
    id: str,
    room_service: RoomService = Depends(),
) -> Optional[FloorplanDetails]:
    return room_service.get_floorplan_by_room_id(id)

@api.get("/{id}/boundaries", response_model=str, tags=["Rooms"])
def get_boundaries(
    id: str,
    room_service: RoomService = Depends(),
) -> str:
    return room_service.get_boundaries_by_room_id(id) 


@api.get("/{id}/circle_tables", response_model=list[CircleTableDetails], tags=["Rooms"])
def get_circletables(
    id: str,
    room_service: RoomService = Depends(),
) -> list[CircleTableDetails]:
    return room_service.get_circletables_by_room_id(id)


@api.get(
    "/{id}/rectangle_tables", response_model=list[RectangleTableDetails], tags=["Rooms"]
)
def get_rectangletables(
    id: str,
    room_service: RoomService = Depends(),
) -> list[RectangleTableDetails]:
    return room_service.get_rectangletables_by_room_id(id)
```

## 2. Floorplan Entity

The Floorplan Entity represents the structural layout of a room, possessing its boundaries and table layouts. It is foundational for organizing seating arrangements. The rectangle and circle table entities are also essential as they hold the dimensions and coordinates of the tables which make up the floorplan for a given room. Below are the fields and relationships declared for the Floorplan Entity, Rectangle Table Entity, and Circle Table Entity. THese entities can be found within `backend/entities/floorplan`.

```py3
class FloorplanEntity(EntityBase):
    """Entity for floorplans for rooms under XL management."""

    __tablename__ = "floorplan"

    id: Mapped[int] = mapped_column(Integer, primary_key=True)
    boundaries: Mapped[str] = mapped_column(String)
    room_id: Mapped[int] = mapped_column(String, ForeignKey("room.id"), nullable=True)

    # relationships
    room: Mapped["RoomEntity"] = relationship("RoomEntity", back_populates="floorplan")
    circle_tables: Mapped[list["CircleTableEntity"]] = relationship(
        "CircleTableEntity", back_populates="floorplan"
    )
    rectangle_tables: Mapped[list["RectangleTableEntity"]] = relationship(
        "RectangleTableEntity", back_populates="floorplan"
    )
```

### 2.1. Rectangle Table Entity

```py3
class RectangleTableEntity(EntityBase):
    """Entity for rectangular tables within floorplans under XL management."""

    __tablename__ = "rectangle"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    x: Mapped[str] = mapped_column(String)
    y: Mapped[str] = mapped_column(String)
    width: Mapped[str] = mapped_column(String)
    height: Mapped[str] = mapped_column(String)
    fill: Mapped[str] = mapped_column(String)

    floorplan_id = mapped_column(Integer, ForeignKey("floorplan.id"))
    floorplan = relationship("FloorplanEntity", back_populates="rectangle_tables")
```

### 2.2. Circle Table Entity

```py3
class CircleTableEntity(EntityBase):
    """Entity for rectangular tables within floorplans under XL management."""

    __tablename__ = "circle"

    id: Mapped[int] = mapped_column(Integer, primary_key=True, autoincrement=True)
    cx: Mapped[str] = mapped_column(String)
    cy: Mapped[str] = mapped_column(String)
    radius: Mapped[str] = mapped_column(String)
    fill: Mapped[str] = mapped_column(String)
    floorplan_id: Mapped[int] = mapped_column(
        Integer, ForeignKey("floorplan.id"), nullable=True
    )

    # relationships
    floorplan: Mapped["FloorplanEntity"] = relationship(
        "FloorplanEntity", back_populates="circle_tables"
    )
```

# Underlying Database / Entity-Level Representation Decisions

Our current underlying database structure has changes to the following entities: room, floorplan, circle_table, and rectangle_table. There is a one to one relationship between room and floorplan, so that a user can retrive a floorplan given a room id. There are one to many relationships between floorplan and circle_table and rectangle_table so that a user can retrieve circle and rectangle tables for a specified room given a floorplan or room id. In our `backend/test/services/coworking/floorplan_data.py`, here is how we connected our floorplan to our rooms, specifically the CSXL floorplan: 
```py3
floorplan_1 = FloorplanDetails(
    id=1,
    boundaries="0 0 150 170",
    room=the_xl.to_room(),
    circle_tables=[],
    rectangle_tables=[],
)
```

# Technical and User Experience Design Choices

## Technical Decisions

### Seat Reservable Attribute
Currently, the Seat model in the back-end is formatted as such: 
```py3

class Seat(SeatIdentity, BaseModel):
    title: str
    shorthand: str
    reservable: bool
    has_monitor: bool
    sit_stand: bool
    x: int
    y: int

```
Therefore, our initial decision was that, after each seat was reserved, we would make the reservable attribute of that seat false. We would then use this attribute's value in our front-end to determine which seats would be grayed out in the seat map widget.

However, we soon realized that, in order for the seat's color to turn to its original color after the reservation became inactive, we would have to somehow manually turn the reservable attribute true or change existing reservation functionality. Instead, we decided to work with the ngOnChanges() function within the dropin-availability-card-widget.ts. This widget keeps track of the currnet seat availability and we used the existing for loop to push each available seat to the CsxlSeatMapService's availableSeatsFromBackend array. This array was then used in our seat map widget to keep track of the seats that are not currently reserved.  
```py3
  ngOnChanges(changes: SimpleChanges): void {
    this.seat_availability = changes['seat_availability'].currentValue;
    this.categories = this.initCategories();
    for (let seat of this.seat_availability) {
      this.mapService.getAvailableSeats(seat.id); // This was the line we added to keep track of the IDs of available seats
      if (seat.has_monitor) {
        if (seat.sit_stand) {
          this.categories[STANDING_BENCH].push(seat);
        } else {
          this.categories[SITTING_BENCH].push(seat);
        }
      } else {
        this.categories[COLLAB_AREA].push(seat);
      }
    }
  }
```

## User Experience Decisions

### Seat Map Location

For our seating map widget, we were originally going to project it onto the main coworking page, similar to how the table showing available seat types. However, our team decided it would be more beneficial to put it in a new component to prevent slowing down the pages compile time. Furthermore, we decided that there would be too much information on one page if the room reservation, CSXL seat map, and operating hours panel were all to be shown together. Therefore, we made the map navigable from the reserve seats button located at the top of the drop-in table on the coworking page which routes the user to the map.

### Widget Color Schema

For our interactive seating chart, we wanted to continue the color scheme of the existing coworking components, especially in room reservation. By doing this we made a legend the consist of the seat types and its respective color. We chose green, yellow, and blue to represent sitting monitors, standing monitors, and communal areas respectively. We chose grey to denote reserved seats so that it would match the floorplans color scheme and we also chose red to denote selected seats for bright emphasis. We thought it was important that our feature's UI represented the current CSXL's color schema. 

# Development Concerns

## Getting Started

To get started on the Seat Map Widget, you must navigate to frontend/src/app/coworking/widgets/seat-map. This widget is the main part of our project as it holds the bulk of our frontend information for displaying and retrieving data for the widget. 

## Tour of Files

1. Seat-Map Component (frontend/src/app/coworking/seating-reservation/csxl-seat-map): This component calls the seat-map widget discussed above to be displayed in the coworking component. Moreover, there is also a csxl-seat-map service that helps in retrieving data from the back end in order to display a floormap.
2. Coworking Reservation Card (frontend/src/app/coworking/widgets/coworking-reservation-card: Instead of hardcoding the first seat of the seats reserved to only display one seat reserved, we now traverse through the list of seats reserved and display all the seat types and ID's in coworking confirmation card.
3. Room Service (backend/services/room.py): This contains conversions of entity data into pydantic models so that the Room API can access this information. 
4. Room API (backend/api/room.py) This file contains all API methods using Room Service in order to be able to fetch crucial floorplan data to the frontend.
5. Models (backend/models): Inside this folder, you will find the Pydantic models for Floorplan, Circle_Table, Rectangle_Table, and Room
6. Entities (backend/entities): Inside this folder, you will find the Entities for Floorplan, Circle_Table, Rectangle_Table, and Room. The entities for floorplan, circle tables, and rectangle tables should be stored within a folder named floorplan.
   
## Concerns

1. Be able to understand how the seat-map widget is connected to preexisting reservation logic within the CSXL code base.
2. Understand the relationship between the Entities we created and how to add demo data within test/services.
3. Understand how we connect the open operating hours and the current seat availability of the CSXL to modify our UI (such as our ReserveSeats button and the greying out of available seats)
