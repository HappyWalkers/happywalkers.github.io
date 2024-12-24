---
title: Smart Home Systems

article_header:
  type: cover
  image:
    src: https://s2.loli.net/2022/12/29/uKdXVlOvp6iShqC.png

tags: Embedded Systems
cover: https://s2.loli.net/2022/12/29/uKdXVlOvp6iShqC.png
coverWidth: 1200
coverHeight: 750
---


# Introduction

<div class="mdui-video-container">
    <iframe src="//player.bilibili.com/player.html?aid=684730157&bvid=BV1BU4y197dp&cid=738874866&page=1" scrolling="no" border="0" frameborder="no" framespacing="0" allowfullscreen="true"> </iframe> // the embedded code
</div>

Our work is an overall system, constructing a one-stop smart home solution from different perspectives, mainly including three modules: smart door control, smart home appliance control, and active health monitoring.

# Smart Gate Control
Smart Gate Control is a face recognition system based on computer vision technology and deep learning. Its core is a Raspberry Pi 4B and an external camera. We trained a residual network for face recognition, and implemented a graphical user interface using PyQt, allowing users to complete face recognition with just a tap of a button, allowing the automatic control system to open the door.

# Smart Appliance Control
Smart home appliance control is a user-friendly interactive system developed based on embedded technology. Its core is a STM32 development board, which provides a rich interface as our development platform. We use a touch display screen to provide a graphical user interface, and send signals to control home appliances through the user’s clicks. The current demo realizes the switching of curtains and lights. We use STM32F429 to control the stepper motor to drive the curtain movement. The user can realize the opening and closing of the curtain by pressing the button or touching the button on the touch screen. At the same time, the GPIO of STM32F429 is used to control the relay corresponding to the light, so that the user can remotely control the indoor lighting when touching the capacitive LCD touch screen.

# Robot-following health monitoring
The robot-following health monitoring module senses the strength of the signal through the radar and other sensing devices equipped with the smart home robot, locates the human body, identifies obstacles, and dynamically plans the path. Realize the automatic follow-up of the care object. The automatic follow function of the smart home robot can locate the location of the caretaker in real time, and record the movement of the caretaker with a camera. Provide better security for vulnerable groups living alone. In the process of realizing automatic follow-up, the care module adopts the yolo framework of deep learning to realize the posture detection of the followed target and judge whether the care target has fallen. Studies have shown that the total case fatality rate of the elderly who falls is 5 times higher than that of the elderly without falls, and the case fatality rate of those who cannot stand up 1 hour after the fall is even twice as high. In the process of caring for vulnerable groups living alone, timely monitoring of falls is of great significance. Through the automatic follow-up and fall detection functions in the nursing module of the smart home robot, the risk of death or injury caused by the accidental fall of the elderly can be greatly reduced. In short, the automation and intelligence of home life is the current development trend of information appliances. I believe that this system will definitely arouse people’s interest and have a wider range of application scenarios after being improved.




