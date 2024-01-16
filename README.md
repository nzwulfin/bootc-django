# django-demo

Use ansible to create a Django environment on top of a bootc based OS with the correct pacakges to support it


The bootc container only contains the OS level mods required for this Django layer to become a functional app environment once configured at run time

The first set of ansible playbooks contains 'generic' customizations that wire up the django environment according to local standards in prepartion for landing the application later

The second set of ansible playbooks come from a represent run time deployment of the application specific configurations